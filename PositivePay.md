# Positive Pay Automated Delivery — Architecture

## Overview

This system automatically generates a Positive Pay file for each organization on a configurable
daily schedule and delivers it to that organization's financial institution's Positive Pay service.

### Supported Financial Institutions

| Institution | Delivery Method |
|---|---|
| Wells Fargo | SFTP / HTTPS (via Azure Logic App) |

## Service Flow

### For Administrators

Once an hour, on the hour, the system wakes up and checks whether any organization is due to have
a Positive Pay file sent to their bank. If one is due, the system automatically builds the file and
transmits it to the financial institution's Positive Pay service — no manual steps required. The
schedule for each organization is configured in a database and can be adjusted independently per
organization.

### For Programmers

1. **Trigger** — The `WFPositivePay` Azure Function fires on an hourly timer (every hour on the
   hour). It queries a database table to identify any organizations whose Positive Pay file is due
   to be generated.

2. **Fan-out** — For each organization that is due, the function publishes a message to an Azure
   Service Bus topic. All TYF web applications subscribe to this topic.

3. **Document generation** — Each TYF web application evaluates the incoming message to determine
   whether it is responsible for that organization. If it is, the application builds the
   financial institution's Positive Pay document.

4. **Transmission** — The generated document is handed off to the `WFPositivePayTransmitted`
   Azure Function, which is responsible for sending the document to the financial institution's
   Positive Pay endpoint and recording the result.

## Flow Diagram

## Azure Services Used

### Azure Functions

- **WFPositivePay** — Triggered on a configurable timer; determines which organizations are due to run and initiates the Positive Pay file generation process.
- **WFPositivePayTransmitted** — Handles post-transmission processing after a Positive Pay file has been delivered to the financial institution.

### Azure Service Bus

Routes messages between Azure Functions and the downstream processing components. Decouples file generation from transmission and allows each step to scale and retry independently.

### Azure Key Vault

- **wfpp-safe-transmissions** — Stores the credentials and connection secrets used when transmitting Positive Pay files to financial institutions.

## Multi-Location / Multi-Account Support (Stacked Files)

**Every organization has one or more locations** — there is no separate "single-location" path in
the data model or build logic. An organization like DistrictBeta that happens to have only one
location is simply represented as a single row; DistrictAlpha, with several locations, has multiple
rows. This location and account-number data **already exists today** in the **Organization
database** — the same database the TYF apps already connect to — so no new table needs to be
created for this. Each location has its own Wells Fargo account number for Positive Pay purposes.
Wells Fargo's ARP Inbound Specification supports this via a **"stacked" multiple-accounts format**:
each location's records form their own block (with that location's account number and its own
control total), and those blocks are concatenated into a **single file per organization** for one
Wells Fargo submission — whether that file ends up containing one block or many.

### Data model (existing)

The `Locations` table already exists in the **Organization database** used by the TYF apps, scoped
under each organization:

| Field | Notes |
|---|---|
| `LocationId` | Primary key |
| `OrgId` | Foreign key to the organization |
| `AccountNumber` | Existing column in the Organization database. Current encryption-at-rest state to be confirmed — if not already encrypted, this should be addressed given it's a live bank account number (see security note below) |
| `Active` | Locations can be turned on/off by the customer without deleting history |
| `DisplayOrder` | Defines the order location blocks are stacked in the output file, per WF's field spec |

### Build logic (updated)

The Service Bus message to the TYF app still carries just `OrgId` — no change to the fan-out
design. On receipt, the TYF app now:

1. Fetches all **active** locations for that org from the **Organization database** (existing,
   already used by the TYF apps) — this always returns one or more rows; there's no special case
   for a "single-location" organization.
2. Calls the external API and builds a validated data block **per location**, using that
   location's own account number and its own control total, per WF's field spec.
3. **Stacks** the location blocks, in the defined order, into a single file. For an organization
   with one location, this naturally produces a file with a single block — the same code path
   handles both cases without branching.
4. Runs an **overall** file validation (total record count, sum of per-location control totals)
   before handing the file to the Logic App for delivery.

### Partial failure handling

If one location's data fetch or validation fails, the org's file should **fail closed by
default** — do not silently submit a file missing that location's block. A missing location means
legitimate checks for that location won't be recognized as authorized by Wells Fargo, which is a
real operational risk, not just a data-quality one. Alert a human to decide whether to hold the
whole file, retry the failing location, or (only if explicitly approved) submit a partial file with
that location visibly flagged. This behavior can be configurable per organization, but "hold and
alert" is the safer default.

### Idempotency and visibility

The run-log key stays `(OrgId, RunDate)` since delivery is still one file per organization. A
**location-level status table** is added alongside it, recording success/failure per location
within that organization's run — otherwise a partial failure inside a stacked file is invisible in
the org-level log.

### Security: account numbers are sensitive existing data — confirm current protections

An account number tied to check payments carries the same risk profile as a credential — a wrong
value doesn't just break a report, it can misdirect real payments. Since this data already exists
in the Organization database and is already being supplied by customers today, the following should
be **confirmed as already in place, and remediated if not**:

- **Encrypted at the column level** in the database, not stored as plain text — worth confirming
  this is the current state; if not, this is a gap worth closing given what the field controls.
- **Every read (at file-build time) and every write (customer update) is audit-logged** — who
  supplied or changed it, when, and which component retrieved it.
- **Changes to an account number go through a maker-checker approval step** before taking effect
  in a production file — a second set of eyes catches a data-entry error before it reaches Wells
  Fargo, rather than after.
- **Whatever interface customers currently use to supply this data** should enforce the same
  access scoping (an organization's admins can only see/edit their own locations) and should not
  allow direct database access. If an admin interface doesn't already exist or doesn't meet this
  bar, that's a gap worth tracking separately from this project's scope.

## Key Design Decisions

1. **Service Bus Topic (not Queue)** — three independently deployed/owned codebases each need their own copy of relevant messages; SQL filters route without any app needing to inspect-and-discard.
2. **Schools processes concurrently** — no session/ordering constraint required on `sub-schools`; concurrency is capped deliberately (`MaxConcurrentCalls` / host.json) to protect the shared upstream API and any shared DB connection pool.
3. **Delivery centralized in one Logic App** — WF credentials, connection setup, host-key/network allowlisting, and retry policy live in exactly one place rather than being duplicated across three codebases. Any change to WF's protocol is a single update.
4. **HTTP handoff, with Blob as audit trail (not the trigger)** — TYF apps call the Logic App directly and get a synchronous delivery confirmation to write into their Run Log, while independently persisting the file to Blob Storage for audit regardless of delivery outcome.
5. **Idempotency enforced at the DB level** — unique constraint on `(OrgId, RunDate)`, not just an application-side check-then-write, since Service Bus is at-least-once delivery and Schools processes concurrently.
6. **Retry ownership lives in the Logic App** — the Logic App's WF connector action retries on failure; TYF apps do not independently retry the same delivery, to avoid duplicate files reaching WF.
7. **Correlation ID generated at the Function** — carried through the Service Bus message, the TYF app's processing, and the Logic App call, so a single org's run is traceable end-to-end across three separately owned codebases in Application Insights.

## Why Not Just Do It All Inside the TYF App? (Plain-Language Rationale)

This section is for anyone looking at the diagram for the first time and thinking: *"Why isn't
this just one app that calls the API and sends the file to the bank? Why all the extra pieces?"*
It's a fair question — the simplest possible version really would just be one app doing everything
in a straight line. Here's why we didn't build it that way, explained one piece at a time.

### "Why can't the TYF app just have the schedule logic built in?"

DistrictAlpha, Schools, and DistrictBeta run from the **same codebase**, but they are deployed
**independently and on different dates** — for example, Schools might be running the version
deployed 2026.07.15, while DistrictAlpha is still running the version deployed 2026.06.12. That gap is
completely normal and expected; it just means each app's deployment schedule is its own, separate
from the others.

If "when do we run" lived inside that codebase, changing an organization's run time would mean
changing code and pushing a new deployment to that specific app — and because deployments happen
on independent schedules, a schedule change might have to wait for that app's next release window,
or force an out-of-cycle deployment just for a time change. Pulling scheduling out into one small,
central configuration store means a schedule change is just a **configuration update** — it takes
effect immediately, for any organization, without needing to wait on or coordinate with any app's
deployment schedule.

### "Why not have the TYF app call the other TYF apps directly, or just run everything in one app?"

We could combine DistrictAlpha, Schools, and DistrictBeta into a single application. We chose not to
because they're **owned and evolve independently** — different levels of complexity (Schools
handles many organizations, the others handle one), potentially different release schedules, and
possibly different teams. Keeping them separate means a change or a bug in one doesn't require
retesting or redeploying the others. The Service Bus topic in the middle is simply the "mail
routing" that gets each organization's task to the right app — nothing more exotic than that.

### "Why not have each TYF app just send the file straight to Wells Fargo itself?" (the security question)

This is the most important one, and it's really a security and risk question, not a technical
preference.

**The core idea: the fewer places that can talk to the bank, the smaller the risk.**

If each of the three TYF apps had its own direct connection to Wells Fargo, that means:

- **Three separate places holding bank credentials** (or three separate places pulling them from
  Key Vault), instead of one. Every additional place a credential *can* be used is another place
  it can be misused, misconfigured, or exposed — by a bug, a misconfigured setting, a compromised
  dependency, or simple human error.
- **Three separate places to network-allowlist with Wells Fargo.** Banks typically require a
  known, fixed set of IP addresses or endpoints permitted to connect. Three apps means three
  things to configure, monitor, and keep in sync if Wells Fargo ever changes requirements.
- **Three separate places that could accidentally send a duplicate or malformed file** to a bank
  Positive Pay feed, if any one app has a retry bug. A malformed or duplicate Positive Pay file is
  not a cosmetic bug — it can cause legitimate checks to be flagged or rejected, or fraudulent ones
  to slip through, so this connection is treated with the same level of care as the credentials
  themselves.
- **Three separate places for an auditor or security reviewer to check**, every time there's a
  security review, an incident, or a compliance audit ("who can send data to Wells Fargo, and how
  do we know it's only sending what it should?").

By centralizing that one connection in a single Logic App:

- There is **exactly one place** that holds the Wells Fargo credentials, one place that's
  allowlisted with the bank, one place with the retry logic that prevents duplicate files, and one
  place to review when it's time for a security audit.
- The TYF apps still do all the actual *work* — pulling data, building the file, validating it —
  they just hand the finished, validated file to the one component whose entire job is "talk to
  Wells Fargo, safely, and report back whether it worked." That's the same principle as a company
  routing all its outgoing wire transfers through one authorized finance system rather than
  letting every department wire money independently — it's not about mistrust of any one team,
  it's about keeping a sensitive external connection contained and easy to reason about.
- If Wells Fargo ever changes their connection requirements (a new encryption method, a new host,
  a new protocol), **one component is updated**, instead of coordinating a change across three
  separately owned applications at the same time.

### "The customer just wants to type in an account number per location — isn't that just ordinary config data like everything else?"

Not quite — even though this data already exists today in the Organization database, it's a
different kind of data from a schedule time or an org name: it's a **bank account number tied to
real checks and real money**. A typo in a run time means a report runs an hour late. A typo in an
account number means Wells Fargo could reject legitimate checks for that location, or — in the
worst case — payments end up validated against the wrong account. So this field deserves the same
protections a credential would get, and it's worth confirming these are actually in place today
rather than assuming they are just because the data has existed for a while:

- **Encrypted in the database**, the same way a password would be — not stored as plain, readable
  text like a schedule time is.
- **Every time it's read or changed, that's logged** — so if something ever goes wrong, there's a
  clear record of who entered it, when, and which part of the system used it.
- **A second person reviews any change before it goes live** (maker-checker) — the same kind of
  control a bank itself uses internally before a change to payment routing takes effect, so a
  single typo from one admin can't silently reach a live Positive Pay file.

None of this is about not trusting the customer — it's the same level of care Wells Fargo itself
applies to account data, and it's standard practice anywhere a system holds "a number that controls
where money goes," whether that data is brand new or has existed for years.

### "So is this all just extra complexity for complexity's sake?"

No — each additional piece exists to solve one specific problem that would otherwise become a real
risk or a real maintenance headache at scale (multiple organizations, multiple teams, a regulated
banking connection, credentials that must never live in application code). The diagram looks more
complex than "one app does everything," but each box is there to keep a sensitive piece of the
process — scheduling, org ownership, or the bank connection itself — isolated, auditable, and
changeable without having to touch or redeploy the other pieces.

## Optional Addition: Datadog as a Monitoring Layer

Application Insights / Azure Monitor is the baseline observability design above and is sufficient
on its own. If the team wants (or already standardizes on) Datadog — for example, to get one
unified monitoring view alongside non-Azure systems — Datadog can consume Azure Monitor and
Application Insights data through a few supported paths, without replacing Application Insights:

- **Datadog's native Azure integration** — pulls metrics and Activity Log data directly from Azure
  Monitor via API, no agent required on individual resources.
- **Diagnostic Settings → Event Hub → Datadog Log Forwarder** — Azure resources in this design
  (Function App, Logic App, Service Bus, the databases) can stream logs/metrics via Diagnostic
  Settings to an Event Hub, which Datadog's Azure Log Forwarder (a Datadog-provided Azure Function)
  picks up and ingests.
- **Application Insights traces specifically** — instrument the apps with OpenTelemetry and
  dual-export to both Application Insights and Datadog APM, so distributed tracing (e.g., the
  `CorrelationId` per org run) shows up in both, or forward App Insights data through the Event Hub
  path above.

This would be an **addition on top of**, not a replacement for, the Application Insights/Azure
Monitor design described elsewhere in this document — Application Insights remains the source of
the telemetry; Datadog becomes an additional consumer of it. Exact setup steps and currently
supported data types should be confirmed against Datadog's current documentation before
implementation, as integration details evolve.

## Open Items to Confirm

- Exact WF Positive Pay delivery protocol (SFTP with PGP vs. HTTPS API) — determines which Logic App connector to use and whether encryption needs to happen before or during the Logic App action.
- WF's tolerance for concurrent connections/sessions from a single sender (relevant given Schools may deliver multiple orgs' files around the same time).
- Auth mechanism on the Logic App's HTTP trigger (Azure AD/Easy Auth vs. API Management) to ensure it isn't reachable by anything other than the three TYF apps.
- Network path from the Azure Function and the three TYF apps to the `WFPositivePay` database, since it's hosted on our own cloud server rather than Azure SQL — needs the same kind of secured connectivity (VPN, ExpressRoute, or firewall-allowlisted access with encryption in transit) already called out for the off-Azure external data API. Worth confirming whether the same network path can be reused for both, or if they require separate configuration.
- Current state of the `AccountNumber` column and its supply mechanism in the existing Organization database: is it encrypted at rest today, is read/write access already audit-logged, and does an admin interface already exist for customers to manage their own locations? Each of these should be confirmed as either already in place or tracked as a remediation item — the same way the `appsettings.json` credential exposure is tracked above.
- **Known exposure:** the Azure Function and the three TYF apps currently store credentials (external API, database) in `appsettings.json` rather than Key Vault. This is a real, acknowledged gap — anyone with access to the deployed app's file system or config can read these values in plain text, and they're at risk of being checked into source control or deployment artifacts. Remediation is to migrate these apps to pull secrets from Key Vault (via Managed Identity, once available on the hosting plan) the same way the Logic App already does for its WF credentials. This should be tracked and prioritized independently of the rest of this design — it's a change to existing apps, not something new introduced by this project.
