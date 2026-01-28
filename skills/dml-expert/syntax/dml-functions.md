# DML Functions and Modifiers Reference

Complete reference for DML functions and expression modifiers.

## Core Concepts

### 1. Dimension and Measure References

**Basic References:**
```yaml
{measure_name}      # Reference a measure
{column_name}       # Reference a column in the current dataset
```

**Cross-Dataset References:**
```yaml
{DATASET_NAME}.column_name                    # Via matching column names (implicit join)
pk({col1}, {col2}).{DATASET_NAME}.column_name # Explicit join columns
pk({col1}, {col2}).table_name(pk1, pk2).field # Direct table reference
```

### 2. Measure Modifiers

Modifiers adjust how measures are calculated: `{measure_name: modifier}`

**Modifier Types:**
- **Dimension modifiers** (+d, -d, dimension) - Adjust aggregation level
- **Filter modifiers** (+f) - Add filtering conditions
- **Time modifiers** (ytd, qtd, ya, pp, pop, yoy) - Time-based calculations

**Modifier Chaining:**
Modifiers apply left-to-right:
```sql
AGG({sales: {ytd, ya}})  -- Year-to-date value from last year
AGG(sales, 'ytd', 'ya')  -- Equivalent syntax
```

---

## Dimension Modifiers

### +d (Drill Down)
Add a dimension to the aggregation:

```sql
-- Drill down to product level
{sales: {+d: product}}

-- Usage in formula
AGG({sales: {+d: transaction_id}})  -- Sales per transaction
```

**Note**: After drilling down, you must aggregate back to current dimension level.

### -d (Roll Up)
Remove a dimension from aggregation (less common).

### dimension
Specify exact dimensions for aggregation:

```sql
{sales: {dimension: [customer, product]}}
```

---

## Filter Modifiers

### +f (Add Filter)
Add filtering conditions to measure calculation:

```sql
-- Simple filter
{sales: {+f: {account_type: 'Income'}}}

-- Multiple values
{debit: {+f: {account_type: ['Expense', 'Other Expense']}}}

-- Complex condition (use quotes)
{expense: {+f: "instr(account, 'Food & drink') > 0"}}

-- Null check
{number_of_checks: {+f: 'customer is not null'}}
```

**Syntax Rules:**
- Dictionary format: `{+f: {column: value}}` or `{+f: {column: [val1, val2]}}`
- String format: `{+f: "SQL condition"}` (requires quotes, escape internal quotes)

---

## Time Modifiers

### Period-to-Date Modifiers

**ytd** (Year-to-Date)
```sql
AGG(sales, 'ytd')  -- Cumulative sales from start of year to current period
```
Supported time dimensions: day, month, quarter

**qtd** (Quarter-to-Date)
```sql
AGG(sales, 'qtd')  -- Cumulative sales from start of quarter
```
Supported time dimensions: day, month

**mtd** (Month-to-Date)
```sql
AGG(sales, 'mtd')  -- Cumulative sales from start of month
```
Supported time dimensions: day

**mat** (Moving Annual Total)
```sql
AGG(sales, 'mat')  -- Rolling 12-month total
```

### Comparison Modifiers

**ya** (Year Ago)
```sql
AGG(sales, 'ya')  -- Sales from same period last year
```

**pp** (Previous Period)
```sql
AGG(sales, 'pp')  -- Sales from previous period (month/quarter/year)
```

**ppp** (Period Before Previous)
```sql
AGG(sales, 'ppp')  -- Sales from two periods ago
```

**p3m** (Previous 3 Months)
```sql
AGG(sales, 'p3m')  -- Sales from last 3 months including current
```

**p6m** (Previous 6 Months)
```sql
AGG(sales, 'p6m')  -- Sales from last 6 months including current
```

### Period Values

**pov** (Period Opening Value)
```sql
AGG(balance, 'pov')  -- Balance at start of period
```

**pcv** (Period Closing Value)
```sql
AGG(balance, 'pcv')  -- Balance at end of period
```

### Growth Rate Modifiers

**yoy** (Year-over-Year Growth)
```sql
AGG(sales, 'yoy')  -- Returns decimal: divide({sales}, {sales: ya}) - 1
```
Returns: 0.15 for 15% growth

**pop** (Period-over-Period Growth)
```sql
AGG(sales, 'pop')  -- Returns decimal: divide({sales}, {sales: pp}) - 1
```
Returns: 0.10 for 10% growth

**⚠️ Important Distinction:**
- `pp` returns the **previous period value**
- `pop` returns the **growth rate** (decimal)

---

## DML Functions

### AGG() - Aggregate Function

Reference measures in queries:

```sql
-- Basic usage
AGG(measure_name)

-- With modifiers (two equivalent syntaxes)
AGG(measure_name, 'modifier1', param1, 'modifier2')
AGG({measure_name: {modifier1: param1, modifier2}})

-- Examples
AGG(sales)
AGG(sales, 'ytd')
AGG(sales, 'ytd', 'ya')
AGG({sales: {+d: product}})
AGG({sales: {+f: {customer: 'John Doe'}}})
```

**Auto-aliasing**: `AGG(sales)` automatically aliases to `sales` - don't add redundant alias.

### DIVIDE() - Safe Division

Prevents division by zero:

```sql
DIVIDE(numerator, denominator)
-- Equivalent to: numerator * 1.0 / NULLIF(denominator, 0)
-- Returns NULL when denominator is 0
```

**Examples:**
```sql
DIVIDE({sales}, {number_of_transactions})  -- Average transaction value
DIVIDE(AGG(sales), AGG(sales, 'pp'))       -- Growth ratio
```

### DISTINCTCOUNT() - Distinct Count

```sql
DISTINCTCOUNT(expression)
-- Equivalent to: COUNT(DISTINCT expression)
```

**Examples:**
```sql
DISTINCTCOUNT({transaction_id})
DISTINCTCOUNT({customer})
```

### Conditional Aggregation Functions

**SUMIF()** - Conditional Sum
```sql
SUMIF(expression, condition)
-- Equivalent to: SUM(CASE WHEN condition THEN expression END)
```

**COUNTIF()** - Conditional Count
```sql
COUNTIF(condition)                    -- Count rows matching condition
COUNTIF(expression, condition)        -- Count non-null values matching condition
-- Equivalent to: COUNT(CASE WHEN condition THEN expression END)
```

**DISTINCTCOUNTIF()** - Conditional Distinct Count
```sql
DISTINCTCOUNTIF(expression, condition)
-- Equivalent to: COUNT(DISTINCT CASE WHEN condition THEN expression END)
```

**AVGIF()** - Conditional Average
```sql
AVGIF(expression, condition)
-- Equivalent to: AVG(CASE WHEN condition THEN expression END)
```

**MAXIF()** - Conditional Maximum
```sql
MAXIF(expression, condition)
-- Equivalent to: MAX(CASE WHEN condition THEN expression END)
```

**MINIF()** - Conditional Minimum
```sql
MINIF(expression, condition)
-- Equivalent to: MIN(CASE WHEN condition THEN expression END)
```

---

## Date Functions (SQLite-style)

### date() Function

```sql
date(expression, modifier1, modifier2, ...)
```

**Modifiers:**

**Relative Offsets:**
```sql
'+N day(s)'      -- Add N days
'-N day(s)'      -- Subtract N days
'+N week(s)'     -- Add N weeks
'-N month(s)'    -- Subtract N months
'+N quarter(s)'  -- Add N quarters
'-N year(s)'     -- Subtract N years
```

**Period Start:**
```sql
'start of week'     -- First day of week (Monday)
'start of month'    -- First day of month
'start of quarter'  -- First day of quarter
'start of year'     -- First day of year
```

**Weekday:**
```sql
'weekday N'  -- Nth day of week (1=Monday, 7=Sunday)
```

**Examples:**
```sql
date(current_date, 'start of month')                    -- First day of current month
date(current_date, 'start of year', '+7 months')        -- August 1st of current year
date(current_date, '-1 year', 'start of year')          -- January 1st of last year
date(current_date, '-3 months', 'start of year', '+3 months')  -- Fiscal year start (April)
date(current_date, '-6 days')                           -- 6 days ago
```

---

## Best Practices

1. **Use AGG() for all measure references** in queries
2. **Chain modifiers left-to-right** for complex calculations
3. **Use DIVIDE() instead of /** to avoid division by zero
4. **Use conditional functions** (SUMIF, COUNTIF) for filtered aggregations
5. **Prefer time modifiers** (ytd, pp, pop) over manual calculations
6. **Quote complex filter conditions** in +f modifier
7. **Don't add redundant aliases** to AGG() calls (auto-aliased)

## Common Patterns

### Year-over-Year Comparison
```sql
SELECT dt_month,
       AGG(sales) as current_sales,
       AGG(sales, 'ya') as last_year_sales,
       AGG(sales, 'yoy') as growth_rate
FROM MODEL
WHERE dt_year = date(current_date, 'start of year')
GROUP BY dt_month
```

### Period-over-Period with Custom Calculation
```sql
SELECT AGG(sales) as current,
       AGG(sales, 'pp') as previous,
       DIVIDE(AGG(sales) - AGG(sales, 'pp'), AGG(sales, 'pp')) as growth_pct
FROM MODEL
WHERE dt_month = date(current_date, 'start of month')
```

### Filtered Measure with Drill-down
```sql
SELECT product,
       AGG({sales: {+f: {customer: 'John Doe'}, +d: product}})
FROM MODEL
GROUP BY product
```

