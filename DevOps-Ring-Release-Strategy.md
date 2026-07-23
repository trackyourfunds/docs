# DevOps Ring-Based Release Strategy Guide

*Based on Microsoft's deployment ring model (["How Microsoft develops DevOps"](https://learn.microsoft.com/en-us/devops/develop/how-microsoft-develops-devops)), adapted for our team's 4-site setup and 1-week sprint cadence.*

---

## 1. Core Idea

Instead of shipping one build to everyone at once, builds move through a sequence of **deployment rings**, each one more stable/production-like than the last. A build only advances to the next ring if it survives in the previous one — this catches regressions early, on a small blast radius, before they reach anything resembling production.

| Ring | Our Site | Contents | Purpose |
|---|---|---|---|
| Ring 0 | **Batch0** | Latest nightly build (always) | Bleeding edge / canary — first place any change lands |
| Ring 1 | **Cheryl** | Latest nightly, *gated* on Batch0 smoke test passing | Early validation — confirms the nightly is at least sane before wider exposure |
| Ring 2 | **ReleaseCandidate** | Previous sprint's build | Stabilized candidate, soak-tested for a full sprint |
| Ring 3 | **Schools** | Sprint before ReleaseCandidate (2 sprints old) | Most conservative/stable ring — what our most risk-averse users see |

Key principle: **each ring lags the one before it, and promotion is gated, not automatic.** Cheryl doesn't get the nightly unless Batch0's smoke test passes.

---

## 2. Branching Model: Trunk-Based Development via PRs

**Developers never commit directly to `master`. All changes land via Pull Request.**

This matters because:
- `master` is the single source of truth Batch0's nightly build is built from. Direct pushes make Batch0 (and everything downstream) unpredictable.
- PRs give a review gate *before* code reaches Ring 0 — the cheapest place to catch a mistake, since nothing has been deployed anywhere yet.
- Clean history makes cherry-picking (Section 9) reliable — cherry-picks work best when each commit is small, self-contained, and clearly described.

**Practical flow:**
1. Developer creates a feature/fix branch off `master`.
2. Opens a PR into `master`.
3. At least one other team member reviews it.
4. PR merges into `master`.
5. That night's nightly build picks up the new `master` and becomes the next Batch0 build.

Full picture: **PR → master → Batch0 (nightly) → Cheryl (if smoke test passes) → RC (weekly) → Schools (weekly)**

---

## 3. Sprint Cadence (1 week, starts Sunday)

```
Sun ─────────────────────────────────────── Sun
 │ Sprint starts                               │ Sprint ends / next sprint starts
 │                                              │
 │ Batch0: nightly build every day              │
 │ Cheryl: nightly build (if smoke test passes) │
 │ ReleaseCandidate: still running last sprint  │
 │ Schools: still running the sprint before that│
```

At each Sunday boundary:
1. **ReleaseCandidate is promoted → Schools** (the build that just finished a full sprint as the candidate now becomes the trusted, older ring) — *except during the occasional weeks it's intentionally skipped; see Section 11.*
2. **The build that was stabilizing all last sprint in Cheryl becomes the new ReleaseCandidate.**
3. **Cheryl/Batch0 continue rolling forward nightly** as usual.

A single build's lifecycle:

`Nightly in Batch0` → (smoke test passes) → `Nightly in Cheryl` → (survives a full sprint) → `Promoted to ReleaseCandidate` → (survives a full sprint) → `Promoted to Schools`

This gives a build **two full weeks of soak time** before it ever reaches Schools — a deliberate, Microsoft-style "time in ring = confidence" model.

### Sprint naming convention

Rather than naming sprints sequentially (`Sprint 1`, `Sprint 2`, `Sprint 47`...), name each sprint after its **start date**:

```
Sprint 2026-07-19
Sprint 2026-07-26
Sprint 2026-08-02
```

This is a widely used best practice for a few reasons:
- **Self-documenting** — anyone can tell when a sprint happened without cross-referencing a spreadsheet, and it survives across team reorgs, tooling migrations, or gaps in numbering.
- **Unambiguous across rings** — since RC and Schools are always running a specific past sprint, a date-named sprint makes it immediately clear *which* sprint's build is sitting in which ring (e.g., "Schools is running Sprint 2026-07-05" is clearer than "Schools is running Sprint 45").
- **Sortable and diffable** — dates naturally sort correctly in tags, filenames, and release notes, whereas sequential numbers can drift if a sprint is ever skipped or repeated.

Use the **Sunday start date** of the sprint as the name, matching the cadence above.

### What the administrator does in GitHub.com at each sprint boundary

Every Sunday, one person (the sprint administrator) performs the promotion in GitHub.com. Concretely:

1. **Tag the outgoing builds first** (see Section 8) — tag whatever is currently in Schools as `schools-YYYY-MM-DD` and whatever is currently in RC as `rc-YYYY-MM-DD`, *before* touching anything else. This is the safety net if the promotion goes wrong.
2. **Promote RC → Schools:**
   - Go to the repo on GitHub.com and switch to the `schools` branch.
   - Click **"Compare"** (or navigate to `github.com/<org>/<repo>/compare/schools...release-candidate`).
   - Click **"Create pull request."** Title it something like `Promote Sprint 2026-07-12 (RC) → Schools`.
   - Confirm the diff is what's expected (i.e., exactly last sprint's RC content, no surprises).
   - Click **"Merge pull request"** → **"Confirm merge."**
3. **Promote Cheryl's stabilized build → ReleaseCandidate:**
   - Same pattern: `github.com/<org>/<repo>/compare/release-candidate...cheryl`.
   - Click **"Create pull request,"** title it `Promote Sprint 2026-07-19 (Cheryl) → ReleaseCandidate`.
   - Review the diff, then **"Merge pull request"** → **"Confirm merge."**
4. **Confirm the deploy pipeline picks up each branch's new content** (whether that's an automatic CI/CD trigger on merge, or a manual "Run workflow" click under the repo's **Actions** tab, depending on how deployment is wired up).
5. **Post the two new tag names and the PR links** in the team channel so everyone knows the promotion happened and what's now running where.

Doing the promotion as a PR (rather than a direct branch merge via git) keeps a reviewable audit trail of exactly what changed at each ring, right in GitHub.com.

---

## 4. Promotion Gates

| Gate | From → To | Criteria |
|---|---|---|
| Smoke test | Batch0 → Cheryl | Smoke checklist passes on Batch0's nightly |
| Sprint soak | Cheryl → ReleaseCandidate | No P0/P1 bugs opened against that build during the sprint; smoke checklist stays green each night |
| Release promotion | ReleaseCandidate → Schools | Full sprint with no regressions found in RC |

---

## 5. Weekly Sprint Ceremonies

- **Sunday** — Sprint kickoff + ring promotion (RC→Schools, Cheryl-stable→RC)
- **Daily** — Nightly build + smoke check on Batch0
- **Friday/Saturday** — Quick review: any regressions in Cheryl this week? Any blockers to promoting to RC on Sunday?

---

## 6. Ring Diagram

```
Batch0 (daily nightly)
   │ smoke test gate
   ▼
Cheryl (daily nightly, gated)
   │ 1 week soak
   ▼
ReleaseCandidate (this sprint's stabilizing candidate)
   │ 1 week soak
   ▼
Schools (fully soaked, 2 sprints old)
```

---

## 7. Rollback Path

Each ring is one sprint behind the ring before it, so there's always a known-good fallback one ring back. **Rule: step backward one ring, never skip.**

| If broken | Roll back to | How |
|---|---|---|
| Schools | Previous Schools build (tagged before this week's promotion) | Redeploy that tag |
| ReleaseCandidate | Previous RC (currently sitting in Schools) | Redeploy the build currently in Schools |
| Cheryl | Last known-good nightly | Redeploy Batch0's most recent *passing* nightly (using that night's `cheryl-YYYY-MM-DD` tag — see Section 8) |
| Batch0 | N/A | No rollback needed — this ring is expected to break sometimes; wait for tomorrow's nightly |

**Key practice: tag every build before it's replaced** (see Section 8 for exactly what this means and how to do it).

Rollback is a **demotion**, not a rebuild — redeploying an artifact/tag that already exists, not rebuilding from source. That's what makes it fast (minutes, not a new sprint cycle).

---

## 8. Tagging (GitHub.com)

### What a tag actually is

A **git tag** is a permanent, named pointer to one specific commit — a snapshot label, like a sticky note on a single point in history that says "this exact state of the code is what was running in [environment] on [date]." Unlike a branch, a tag doesn't move forward as new commits land — once created, it always points to that same commit.

This is exactly what rollback needs: a tag captures "this was the exact code running in ReleaseCandidate the moment before we replaced it," so we can get back to it later even after RC has moved on.

### Tagging convention (all three rolling rings)

Tags are created for **all three rings that get replaced regularly** — not just RC and Schools:

- `schools-YYYY-MM-DD`
- `rc-YYYY-MM-DD`
- `cheryl-YYYY-MM-DD` — tag **every night's** Cheryl build that passes smoke test, right before the next nightly replaces it. This gives day-level rollback granularity on Cheryl, not just sprint-level, which matters since Cheryl changes daily instead of weekly.

Batch0 is not tagged — it's expected to break sometimes and simply waits for tomorrow's nightly (see Section 7).

### Creating a tag on GitHub.com

1. Go to the repo → **`< > Code`** tab → branch/tag dropdown → switch to the branch currently deployed to that ring (e.g., `release-candidate`, `schools`, or `cheryl`).
2. Open the **Commits** history for that branch and find the specific commit currently running in that environment (usually the latest one on the branch).
3. Copy that commit's SHA (clipboard icon next to the commit hash).
4. Go to the repo's **Releases** page (`github.com/<org>/<repo>/releases`).
5. Click **"Draft a new release."**
6. Click **"Choose a tag"** → type the new tag name using our naming convention (`schools-YYYY-MM-DD`, `rc-YYYY-MM-DD`, or `cheryl-YYYY-MM-DD`).
7. Click **"Create new tag: [name] on publish."**
8. Under **"Target,"** click the branch dropdown → **"Recent Commits"** → select the exact commit SHA copied in step 3 (important — the tag must land on that specific commit, not just whatever is newest at the time).
9. Add a short description (e.g., "RC build running week of July 20").
10. Click **"Publish release"** (a plain tag without full release notes is fine — a one-line description is enough).

### Viewing existing tags

1. Go to `github.com/<org>/<repo>/tags`.
2. Tags are listed newest first — filter/scan for the naming pattern (e.g., `schools-`, `cheryl-`).
3. Click a tag to see the commit it points to and its description.

### Rolling back using a tag

1. Open the **Tags** page and open the tag to roll back to (e.g., `schools-2026-07-13`).
2. Note the commit SHA it points to.
3. Go to the branch for that ring (e.g., `schools`).
4. GitHub.com's UI doesn't expose a hard-reset button, so the reliable web-only path is:
   - Open the tag's commit page → **"Browse files"** to confirm it's the right state.
   - Use the compare view (`.../compare/schools...schools-2026-07-13`) to review the diff.
   - Create a new branch from the tag, open a PR into `schools`, and merge it. This keeps full history and avoids force-pushing, which also isn't exposed as an easy button on GitHub.com.
5. Redeploy from `schools` once the branch reflects the tagged commit.

This PR-based rollback keeps everything reviewable and avoids anyone needing repo-admin force-push permissions.

### What "tag every build before it's replaced" means, concretely

Every Sunday, right **before** this week's promotion overwrites RC or Schools (and every night, right before the next nightly overwrites Cheryl):

1. Note the commit SHA currently deployed to that ring.
2. On GitHub.com: **Releases → Draft a new release** → tag that commit (`schools-YYYY-MM-DD`, `rc-YYYY-MM-DD`, or `cheryl-YYYY-MM-DD`) → publish.
3. *Then* deploy the new build into that ring.

This leaves a fresh tag each time — "last known-good Schools," "last known-good RC," or "last known-good Cheryl" — captured right before that ring's content is replaced. If the new promotion (or nightly) breaks something, the exact prior commit is already tagged and ready to roll back to, without rebuilding anything.

---

## 9. Cherry-Picking on Schools and ReleaseCandidate

Cherry-picking comes up when a specific fix needs to reach Schools or RC **before** the normal Sunday promotion — e.g., a critical bug is found and you can't wait a week for the next scheduled promotion.

### Rule of thumb

Cherry-picks are for **hotfixes only** — small, isolated, low-risk changes (typically a single commit). They are not a shortcut for promoting a whole sprint's worth of unreleased work early. If a fix touches a lot of code or has ripple effects, let it ride the normal weekly promotion instead.

### Cherry-picking onto ReleaseCandidate

**Using GitHub Desktop:**
1. Open the repo in GitHub Desktop and switch the current branch to `release-candidate`.
2. Click **History** and switch the branch dropdown to `cheryl` (or wherever the fix currently lives) to find the commit.
3. Right-click the commit containing the fix → **"Cherry-pick commit..."**
4. In the dialog, choose `release-candidate` as the target branch and confirm.
5. GitHub Desktop applies the commit onto `release-candidate`; resolve any conflicts it flags in the **Changes** view, then mark them resolved and continue the cherry-pick.
6. Click **"Push origin"** to push `release-candidate` with the new commit.

**Using GitHub CLI (`gh`) + git:**
GitHub CLI itself doesn't have a dedicated cherry-pick command — cherry-picking is a core `git` operation — but `gh` is useful for opening the follow-up PR so the change is reviewable. Typical flow:
```bash
gh repo clone <org>/<repo>
cd <repo>
git checkout release-candidate
git pull origin release-candidate
git cherry-pick <commit-hash>
# resolve conflicts if any, then:
git add .
git cherry-pick --continue
git push origin release-candidate
```
Optionally, do the cherry-pick on a short-lived branch and open a PR with `gh` instead of pushing straight to `release-candidate`:
```bash
git checkout -b hotfix/rc-<short-sha> release-candidate
git cherry-pick <commit-hash>
git push origin hotfix/rc-<short-sha>
gh pr create --base release-candidate --head hotfix/rc-<short-sha> \
  --title "Cherry-pick <short-sha> onto RC" --body "Hotfix for <ticket/description>"
```

7. **Rebuild and deploy to RC.**
8. **Re-run the smoke checklist against RC** before considering it done — a cherry-pick bypasses the normal sprint-long soak, so this is your only safety net.
9. **Log the cherry-pick** (commit hash, reason, who approved it) in the tracking log below.

### Cherry-picking onto Schools

Same process, but higher stakes since Schools is your most stable/visible ring. **Don't cherry-pick straight from Cheryl into Schools — always go through RC first if at all possible**, so the fix gets at least some validation.

**Using GitHub Desktop:**
1. Switch the current branch to `schools`.
2. In **History**, switch to `release-candidate`, find the commit, right-click → **"Cherry-pick commit..."**
3. Choose `schools` as the target, resolve any conflicts (Schools is 2 sprints behind, so drift is more likely here), then push.

**Using GitHub CLI + git:**
```bash
git checkout schools
git pull origin schools
git cherry-pick <commit-hash>
# resolve conflicts, then:
git cherry-pick --continue
git push origin schools
```
Or via a reviewable PR:
```bash
git checkout -b hotfix/schools-<short-sha> schools
git cherry-pick <commit-hash>
git push origin hotfix/schools-<short-sha>
gh pr create --base schools --head hotfix/schools-<short-sha> \
  --title "Cherry-pick <short-sha> onto Schools" --body "Hotfix for <ticket/description>"
```

4. Rebuild and deploy to Schools.
5. Smoke test Schools directly after deploy.
6. Log the cherry-pick the same way as above.

### The tricky part: keeping cherry-picks from getting lost on the next promotion

Because promotions move whole builds forward (RC→Schools, Cheryl-stable→RC), a fix that was cherry-picked directly onto Schools **will get overwritten** the next time RC is promoted into Schools — unless that same fix also exists upstream in RC/Cheryl/`master`.

**Rule: every cherry-pick must also be applied (or already present) upstream**, so it doesn't vanish on the next scheduled promotion. Practically:
- Cherry-pick into Schools → also cherry-pick the same commit into RC and Cheryl if it isn't already there.
- Cherry-pick into RC → also cherry-pick into Cheryl if it isn't already there.
- The fix should exist on `master` before it's considered fully resolved — **Schools should never hold a fix that `master` doesn't also have**, or it will disappear at the next promotion.

### Quick reference: which scenario am I in?

| Question | If yes |
|---|---|
| Does the bug already exist as a fixed commit in `master`/Cheryl/RC? | **Scenario A** — cherry-pick the existing fix forward into Schools (and RC if needed) |
| Is this the first time anyone's fixed it, anywhere? | **Scenario B** — fix via `master` first (normal PR process), then cherry-pick that same commit into RC and Schools immediately |

### Cherry-pick tracking log (template)

| Date | Commit SHA | Description | Applied to | Also backfilled to |
|---|---|---|---|---|
| 2026-07-20 | a1b2c3d | Fix null ref in checkout | Schools | RC ✅, Cheryl ✅ |

---

## 10. Example: A Full Ring Promotion Walkthrough

Assume sprints are named by their Sunday start date, and it's the morning of **Sunday, July 19, 2026**.

**Before promotion, the rings hold:**

| Ring | Currently running |
|---|---|
| Schools | Sprint 2026-07-05 |
| ReleaseCandidate | Sprint 2026-07-12 |
| Cheryl | Nightly builds from Sprint 2026-07-12, stabilized all week |
| Batch0 | Last night's nightly |

**Administrator's steps that morning:**

1. Tag the outgoing Schools build: `schools-2026-07-19` (this is the last snapshot of Sprint 2026-07-05).
2. Tag the outgoing RC build: `rc-2026-07-19` (this is the last snapshot of Sprint 2026-07-12).
3. Open a PR: `compare/schools...release-candidate` → merge → **Schools now runs Sprint 2026-07-12.**
4. Open a PR: `compare/release-candidate...cheryl` → merge → **ReleaseCandidate now runs Sprint 2026-07-19** (the build that stabilized in Cheryl all last week).
5. Cheryl and Batch0 continue rolling forward nightly as usual, starting to accumulate Sprint 2026-07-26's changes.

**After promotion, the rings hold:**

| Ring | Now running |
|---|---|
| Schools | Sprint 2026-07-12 |
| ReleaseCandidate | Sprint 2026-07-19 |
| Cheryl | Nightly builds, starting Sprint 2026-07-26 |
| Batch0 | Tonight's nightly |

Two weeks later, on **Sunday, August 2, 2026**, Sprint 2026-07-19 will make its final move from RC into Schools — completing its two full weeks of soak time.

---

## 11. Skipping ReleaseCandidate → Schools (Optional)

Promoting ReleaseCandidate into Schools every single Sunday is the default, but it is **not mandatory every week**. The team may choose to hold Schools steady on its current build for a few extra weeks — for example:

- Schools is serving a particularly risk-sensitive period (e.g., exam season, a major event, a holiday freeze) where the team wants zero churn regardless of how clean RC looks.
- A recent Schools promotion is still being monitored and the team wants more observation time before layering another change on top.
- There's a known low-priority issue sitting in RC that the team would rather let ride an extra sprint of soak before it becomes the new Schools baseline.

**How this works in practice:**
- **Cheryl → RC promotion still happens every Sunday, without exception.** This keeps Cheryl/RC moving forward and prevents a backlog of unpromoted work from piling up.
- **Only the RC → Schools step is paused.** Schools simply continues running its current build for another sprint (or several).
- Skip a Sunday for RC → Schools by simply not opening the `compare/schools...release-candidate` PR that week — there is no branch action needed to "hold," since Schools is unaffected unless that PR is merged.
- When ready to resume, the next RC → Schools promotion will carry forward **everything accumulated across the skipped sprints at once** — so the diff review that week (Section 3, step 2) will be larger than a normal single-sprint diff. Budget more review time for it.
- Document each skip decision briefly (who decided, why, and for how long) in the same place the cherry-pick log lives, so it's clear later why Schools sat still for multiple sprints.

This flexibility is intentional: the ring model's core promise (Schools is always a known-good, previously-soaked build) still holds even when a promotion is skipped — Schools just soaks a little longer before its next update.

---

## 12. Using VS2022 Publishing

This describes the day-to-day and week-to-week steps for whoever is running builds through Visual Studio 2022's built-in **Publish** feature (right-click project → Publish), rather than through GitHub Actions or another external pipeline. This is the actual workflow currently in use: pull `master`, build, publish to Batch0, then — if the smoke test passes — publish to Cheryl.

### One-time setup

Create **four Publish Profiles**, one per ring, each pointing at that ring's real deployment target (server/site/slot):
- `Batch0`
- `Cheryl`
- `ReleaseCandidate`
- `Schools`

(Right-click project → **Publish** → **New Profile** → repeat for each. Name them clearly so there's no ambiguity about which profile targets which environment.)

### Daily flow: Batch0 → Cheryl

1. Open the solution in VS2022 on the `master` branch. In the **Git Changes** window, confirm the current branch is `master`, then **Pull** to get the latest.
2. **Build** the solution (**Build → Build Solution**, or Ctrl+Shift+B) — confirm it succeeds with no errors.
3. **Publish to Batch0**: right-click the project → **Publish** → select the `Batch0` profile → **Publish**.
4. Run the smoke checklist against Batch0.
5. **If it passes:** publish the same build to Cheryl — right-click → **Publish** → select the `Cheryl` profile → **Publish**. (No new build is needed if both profiles deploy from the same build output; if the profiles are configured with separate build/output paths, rebuild first so Cheryl isn't accidentally served stale output.)
6. **If it fails:** do not publish to Cheryl. Fix forward on `master` via a normal PR (Section 2) — Batch0 will pick up the fix on tomorrow's nightly.
7. Once Cheryl is live, tag that commit (Section 8) — either on GitHub.com, or directly from VS2022: in the **Git Repository** window, right-click the commit → **"Create Tag..."** → name it `cheryl-YYYY-MM-DD` → then right-click the new tag under the **Tags** node → **Push**.

### Weekly flow: promoting to ReleaseCandidate and Schools

RC and Schools each run an **older sprint's build**, not whatever currently sits on `master` — so promoting them isn't "publish what's open," it's "publish a specific past commit." Every Sunday:

**Promoting to ReleaseCandidate:**
1. In the **Git Repository** window, checkout the commit that's been stabilizing in Cheryl all week — either the `cheryl` branch's current tip, or last night's `cheryl-YYYY-MM-DD` tag if you want to be exact about which nightly is promoted.
2. Pull to confirm nothing's missing, then **Build**.
3. **Publish** using the `ReleaseCandidate` profile.
4. Tag the newly-published commit `rc-YYYY-MM-DD` (same tagging steps as above).

**Promoting to Schools:**
1. First, tag the *outgoing* Schools build (`schools-YYYY-MM-DD`) if this hasn't already been done — this captures the rollback safety net before anything is overwritten.
2. In the **Git Repository** window, checkout the commit that's been sitting in ReleaseCandidate for the full sprint that's now ending — this is the same commit tagged `rc-YYYY-MM-DD` from the *previous* Sunday, not the one you just created above in this same session.
3. Pull to confirm, then **Build**.
4. **Publish** using the `Schools` profile.

### Cherry-picks via VS2022

If a hotfix needs to reach RC or Schools outside the normal Sunday promotion:
1. In the **Git Repository** window, right-click the fix commit (on `master` or `cheryl`) → **"Cherry-pick commit..."**
2. Choose `release-candidate` or `schools` as the target branch and confirm.
3. Resolve any conflicts flagged in the **Changes** view, mark them resolved, and complete the cherry-pick.
4. **Build**, then **Publish** using the matching ring's profile (`ReleaseCandidate` or `Schools`).
5. Log the cherry-pick in Section 9's tracking log, same as the GitHub Desktop/CLI methods.

### Important gotcha

VS2022's Publish deploys **whatever is currently checked out in the working directory** — it has no concept of "which sprint should be running where." Before clicking Publish for RC or Schools, always double-check the **Git Changes** window shows the correct branch/commit/tag checked out. Publishing `master`'s latest straight to Schools by mistake would completely break the ring model's soak-time guarantees.

---

## 13. Open Items to Revisit

- Consider whether GitHub CLI or GitHub Desktop should be the team's default for cherry-picks going forward, now that both are documented in Section 9.
- Revisit whether `cheryl-YYYY-MM-DD` tags need to be pruned/retired after some age, since they accumulate daily rather than weekly.
