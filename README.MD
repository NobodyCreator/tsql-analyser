# T-SQL Analyser

A lightweight browser-based tool for detecting common T-SQL performance anti-patterns.

---

## What it detects

### High severity

**`CAST`/`CONVERT` on a column in a filter clause**
Wrapping a column in `CAST()` or `CONVERT()` inside a `WHERE`, `ON`, or `HAVING` clause forces SQL Server to evaluate the function on every row in the table. It cannot use an index seek — it must scan. Remove the wrapper and rewrite the comparison value instead.

```sql
-- Bad
WHERE CAST(o.OrderDate AS DATE) = '2024-06-01'

-- Good
WHERE o.OrderDate >= '2024-06-01' AND o.OrderDate < '2024-06-02'
```

**`NOT IN (SELECT ...)`**
If the subquery returns any `NULL` value, `NOT IN` silently returns no rows at all — a common bug that's hard to spot. It also tends to produce worse query plans than the alternatives.

```sql
-- Bad
WHERE o.StatusID NOT IN (SELECT StatusID FROM tbl_Status WHERE IsActive = 0)

-- Good
WHERE NOT EXISTS (SELECT 1 FROM tbl_Status s WHERE s.StatusID = o.StatusID AND s.IsActive = 0)
```

**Cross-database subqueries**
Subqueries that reference another database (e.g. `reporting.dbo.tbl_Customers`) prevent the optimiser from building a unified execution plan across the boundary. Materialise the result into a temp table first.

---

### Medium severity

**Correlated `EXISTS` subquery**
An `EXISTS` subquery that references a column from the outer query re-executes once per row in the outer result set. On large tables this is expensive. Rewrite as a `JOIN` or materialise the inner set in a CTE.

**`IN (SELECT ...)` — prefer `EXISTS` or `JOIN`**
`IN (SELECT ...)` is often less efficient than `EXISTS` or a `JOIN`. SQL Server sometimes rewrites these automatically but not always an explicit `EXISTS` gives the optimiser more flexibility.

**Other functions wrapping columns in filters**
`YEAR()`, `MONTH()`, `LEFT()`, `UPPER()`, and similar functions applied to a column in a filter clause have the same index-blocking effect as `CAST()`.

**`WITH (NOLOCK)`**
Reads uncommitted data including rows from transactions that may still roll back. Can return inconsistent, duplicate, or missing rows. Rarely the right fix — consider `READ_COMMITTED_SNAPSHOT` at the database level instead.

---

### Low severity

**`SELECT *`**
Selecting all columns prevents covering index usage and transfers unnecessary data. Name only the columns your application needs.

---

## Why I built this

Working across multiple SQL Server environments I kept seeing the same patterns cause the same problems containing slow reports, index scans where seeks should be, silent `NOT IN` bugs. This tool makes it easy to catch them before they hit prod.

---

## Usage

1. Open `index.html` in any browser 
2. Paste a T-SQL query (or click **Load example**)
3. Click **Analyse**
4. Use **Share** to generate a URL with the query pre-loaded
5. Use **Copy results** to export findings as plain text

---

## Limitations

Static analysis only as the tool reads the query as text. It cannot:
- Know your actual schema or index definitions
- Detect implicit type conversions without schema info
- Catch every possible performance issue

Use it as a first-pass assistant check, not a replacement for reviewing execution plans.