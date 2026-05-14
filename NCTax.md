# North Carolina Tax Information

## TrackYourFunds (TYF) Implementation Note

> ?? **TYF uses the Transaction-Level (Invoice) calculation method.**
>
> Tax is computed on the aggregate total of all taxable detail lines in a transaction, and the rounding rule is applied **once** to that combined amount. Individual line items are **not** independently rounded — the single rounded tax figure applies to the transaction as a whole, consistent with the invoice basis election permitted under § 105?164.10.

---

## Reference

[NC General Statutes - Chapter 105, Article 5](https://www.ncleg.gov/EnactedLegislation/Statutes/PDF/ByArticle/Chapter_105/Article_5.pdf)

---

## 105?164.10. Retail Tax Calculation

For the convenience of the retailer in collecting the tax due under this Article, the Secretary must prescribe tables that compute the tax due on sales. The Secretary must issue a separate table for each rate of tax that may apply to a sale. A retailer is not required to collect tax due under this Article based on a bracket system.

*NC General Statutes - Chapter 105 Article 5 46*

In computing tax due under this Article, the tax computation must be carried to the third decimal place and must round up to the next cent whenever the third decimal place is greater than four. A person liable for tax under this Article may elect to compute the tax due on a transaction on an item or invoice basis and the rounding rule is applied to the aggregate tax due.

*(1957, c. 1340, s. 5; 1961, c. 826, s. 2; 1973, c. 476, s. 193; 1991, c. 689, s. 313; 2013?316, s. 5(e); 2014?3, s. 5.1(f); 2017?204, s. 2.9(h).)*

---

## Explanation of Tax Calculation Methods

The statute permits two approaches for computing sales tax on a transaction. The key difference lies in **when** the rounding rule is applied.

### Item-Level (Detail Line) Tax Calculation

In this method, tax is calculated and rounded **individually on each line item** within a purchase order (PO) or check transaction.

**How it works:**
1. For each detail line, multiply the line's taxable amount by the applicable tax rate.
2. Carry the result to the third decimal place.
3. If the third decimal place is **greater than 4**, round **up** to the next cent; otherwise, truncate.
4. The total tax due for the transaction is the **sum of all individually rounded line-item taxes**.

**Example** (7.25% tax rate):

| Line | Amount   | Tax (unrounded) | Tax (rounded) |
|------|----------|-----------------|---------------|
| 1    | $10.00   | $0.725          | $0.73         |
| 2    | $5.00    | $0.3625         | $0.36         |
| 3    | $8.00    | $0.580          | $0.58         |
| **Total** |      |                 | **$1.67**     |

> Each line is rounded independently before being summed.

---

### Transaction-Level Tax Calculation

In this method, tax is calculated on the **aggregate total** of all taxable line items in the transaction, and rounding is applied **once** to that combined amount.

**How it works:**
1. Sum all taxable line item amounts into a single transaction total.
2. Multiply the total by the applicable tax rate.
3. Carry the result to the third decimal place.
4. Apply the rounding rule once to the aggregate tax amount.

**Example** (same 7.25% tax rate):

| Line | Amount   |
|------|----------|
| 1    | $10.00   |
| 2    | $5.00    |
| 3    | $8.00    |
| **Total** | **$23.00** |

> $23.00 × 7.25% = $1.6675 ? third decimal is **7** (greater than 4) ? rounds up to **$1.67**

---

### Why the Two Methods Can Produce Different Results

Because rounding is applied at different stages, the two methods can yield a **1-cent difference** on the same transaction. Item-level calculation accumulates rounding on each line, which can result in a slightly higher total tax than transaction-level calculation where rounding happens only once.

| Method             | Tax Due |
|--------------------|---------|
| Item-level         | $1.67   |
| Transaction-level  | $1.67   |

> In this example the results match, but with different amounts or more line items the totals can diverge by one or more cents.

The statute explicitly allows a business to **elect** either method, but the chosen method must be applied consistently to the transaction (item basis or invoice/transaction basis).

---

## 5-Line Detail Example: Item-Level vs. Transaction-Level

The following example uses a **7.25% tax rate** and demonstrates how the two methods can produce different results on the same purchase order.

### Detail Lines

| Line | Description        | Amount   |
|------|--------------------|----------|
| 1    | Office Supplies    | $12.49   |
| 2    | Printer Paper      | $34.99   |
| 3    | Ink Cartridges     | $27.15   |
| 4    | Stapler            | $8.75    |
| 5    | USB Drive          | $19.99   |

---

### Method 1: Item-Level (Detail Line) Calculation

Tax is computed and rounded on **each line individually**, then the rounded amounts are summed.

| Line | Amount   | Calculation             | Tax (3 decimals) | 3rd Decimal | Rounded Tax |
|------|----------|-------------------------|------------------|-------------|-------------|
| 1    | $12.49   | $12.49 × 0.0725         | $0.905           | 5 > 4 ? ?   | $0.91       |
| 2    | $34.99   | $34.99 × 0.0725         | $2.536           | 6 > 4 ? ?   | $2.54       |
| 3    | $27.15   | $27.15 × 0.0725         | $1.968           | 8 > 4 ? ?   | $1.97       |
| 4    | $8.75    | $8.75 × 0.0725          | $0.634           | 4 ? 4 ? =   | $0.63       |
| 5    | $19.99   | $19.99 × 0.0725         | $1.449           | 9 > 4 ? ?   | $1.45       |
| **Total** |      |                         |                  |             | **$7.50**   |

> Each line is rounded before adding to the total. Line 4 ($0.634) is **not** rounded up because the third decimal is exactly **4**, which is not *greater than* 4.

---

### Method 2: Transaction-Level (Invoice) Calculation

All line amounts are **summed first**, then tax is computed and rounded **once** on the aggregate total.

| Line | Amount   |
|------|----------|
| 1    | $12.49   |
| 2    | $34.99   |
| 3    | $27.15   |
| 4    | $8.75    |
| 5    | $19.99   |
| **Subtotal** | **$103.37** |

> $103.37 × 0.0725 = **$7.494** ? third decimal is **4** (not greater than 4) ? **no round up**

| Aggregate Tax (3 decimals) | 3rd Decimal | Rounded Tax |
|----------------------------|-------------|-------------|
| $7.494                     | 4 ? 4 ? =   | **$7.49**   |

---

### Comparison Summary

| Method               | Tax Due | 
|----------------------|---------|
| Item-Level           | $7.50   |
| Transaction-Level    | $7.49   |
| **Difference**       | **$0.01** |

> This illustrates how the item-level method can produce a **1-cent higher result** due to rounding accumulating across multiple detail lines. Both methods are legally permitted under § 105?164.10, but the business must consistently apply its elected method.
