# Model Query SQL Specification

Complete specification for Model Query SQL.

## Language Overview

Model Query SQL is a data query language using SQL syntax subset for DML data models. It has powerful expression capabilities through DML, so Model Query SQL only needs **single table, single query** - **prohibit subqueries, UNION, or table joins**.

---

## Query Type Determination

Model Query SQL supports two query types: **Detail Queries** and **Aggregation Queries**.

### Priority Order

**Existence check > Extreme value description > Aggregation need > Other types**

### Detail Query Scenarios

Use detail queries when:

1. **Existence Checks**: Questions asking "is there any", "did we", "have we"
   - SELECT must include **core entity fields** (product, customer) + **transaction key fields** (dt_date, transaction_id, raw_amount)
   - Prohibit using only DISTINCT

2. **Extreme Value Record Queries**: Queries with "latest/earliest/most recent/highest/lowest/largest/smallest/most/least expensive" **without dimension grouping**
   - Use `ORDER BY ... LIMIT 1` instead of `MAX()`
   - Example: "latest invoice", "earliest payment" should return specific transaction records

3. **List Queries**: Queries requesting to list, find, or show specific records or name lists
   - Use detail query with DISTINCT

**Key**: Determined by the nature of the result, not whether the query mentions quantity words.

### Aggregation Query Scenarios

Use aggregation queries when:

1. **Extreme Value Analysis**: Queries with extreme value descriptions **requiring dimension analysis**
   - Returns **dimension** + **measure value**
   - Example: "largest expense" by account → `SELECT account, AGG(expense) FROM MODEL GROUP BY account ORDER BY AGG(expense) DESC LIMIT 1`

2. **Statistical Calculations**: Queries requiring totals, averages, or other aggregated results

---

## DML Data Model

- DML data models use datoml expressions to define datasets, column-macros, and measures in YAML format
- Includes all content from `<base_model>` and `<model>`, plus extensions via ```dml``` blocks
- **Column-macros**: Supplement column definitions for datasets (same effect as defining fields directly in dataset)
- **Field/Measure References**: Definitions can use `{field_name}` and `{measure_name}` to reference other fields/measures
- **Access**: Any field and measure can be accessed by name from MODEL table and MODEL.dataset_name table
- **Missing Fields**: When query mentions fields not defined in `<base_model>` or `<model>`, generate corresponding DML definitions and output in ```dml``` block

---

## Data Source Rules

### Aggregation Queries

- **Data Source**: Must query from **MODEL** table
- **Measure Reference**: Use `AGG(measure_name)` function to get measure values; must reference at least one measure via AGG
- **Filter Scope**: WHERE and GROUP BY apply to measures, making measures return calculated values under GROUP BY dimensions after applying WHERE
- **Prohibit Direct Aggregation**: Prohibit using SQL or datoml aggregation functions directly on dataset fields; must reference measures via AGG function
- **Table Structure**: MODEL table contains all fields and measures from model definition

### Detail Queries

- **Data Source**: Must query from **MODEL.dataset_name** table (dataset_name must exist in `<model>` or `<base_model>`)
- **Prohibitions**: Prohibit AGG function, prohibit window functions, prohibit GROUP BY
- **Table Structure**: MODEL.dataset_name table contains all fields (including column-macros) related to dataset_name

---

## General Rules

### Basic Format Requirements

- **Single-line Code**: Strictly generate single-line DSQL code without line breaks, indentation, or comments
- **Prohibit Complex Expressions**: No complex calculations, no table joins, no subqueries
- **Complete Output**: Must output all queried fields
- **AGG Alias Rules**:
  - `AGG(measure_name)` automatically sets alias to measure_name - don't add redundant alias
  - When field expression uses modifiers or formulas, must set alias for the field
  - Example: Use `AGG(sales)` not `AGG(sales) as sales`
  - Example: Use `AGG(sales, 'pop') as sales_pop`
- **Field Naming**: Field names cannot contain spaces; data values can contain any characters

### Minimization Principles

- **Prohibit Adding Unrequested Elements**: Don't add ORDER BY, aliases, DML definitions unless explicitly required or semantically necessary
- **Prohibit Redundant DML Definitions**: When fields/measures already exist in `<base_model>` or `<model>`, don't generate duplicate DML blocks
- **Prohibit Redundant Aliases**: Only set aliases when field uses modifiers/formulas or needs renaming
- **Prohibit Redundant Null Filters**: Prohibit adding non-explicitly-requested null filters for mentioned fields (like `equipment is not null`, `customer is not null`), unless query explicitly requires excluding nulls (e.g., "exclude null customers / remove blanks")

### Concise Expression Priority

**Prefer concise expressions:**

```sql
-- Use this
dt_quarter = date(current_date, 'start of quarter')
-- Avoid this
dt_date BETWEEN date(current_date, 'start of quarter') AND date(current_date)

-- Use this
dt_week = date(current_date, 'start of week')
-- Avoid this
dt_date >= date(current_date, 'start of week') AND dt_date < date(current_date, 'start of week', '+7 days')

-- Use this
AGG(sales, 'pop')
-- Avoid this
AGG(sales) / AGG(sales, 'pp') - 1
```

---

## Field Usage Rules

### Priority Order

1. **First priority**: Use names mentioned in the query as field names
2. **Second priority**: Use synonym fields/measures already defined in examples, `<model>`, or `<base_model>`
3. **Third priority**: Use newly defined column-macros/fields/measures when mentioned names don't exist

### Abbreviation Handling

When query uses abbreviations (e.g., AR for Accounts Receivable), first check if model contains fields with that abbreviation (case-insensitive). If the field directly corresponds to the queried concept, use that field directly rather than reconstructing logic from other fields.

### Column Description Priority

When selected field has `description`, strictly follow instructions in the description. If description marks "internal use only" and specifies alternative field, **must completely replace** original field - don't use original field in WHERE/SELECT.

Examples: `product_service_name` → `product`, `customer_name` → `customer`

---

## Time Handling

### Time Field Naming

- `dt_date`: Day-level granularity
- `dt_week`: Week-level (Monday start)
- `dt_month`: Month-level
- `dt_quarter`: Quarter-level
- `dt_year`: Year-level
- **Principle**: Use first day of period to represent entire period

### Time Filtering (WHERE)

**Format Requirements**: Use first-day format
```sql
dt_year = '2023-01-01'                                    -- Year 2023
dt_month = date(current_date, 'start of month')           -- Current month
dt_quarter = date(current_date, 'start of quarter', '-1 quarter')  -- Last quarter
```

**Past Period Handling**:
- **Past N months/quarters/years**: Usually exclude current period (incomplete data)
  - Example: Past 12 months → `dt_month BETWEEN date(current_date, 'start of month', '-12 months') AND date(current_date, 'start of month', '-1 month')`
- **Past N days**: Must include current day
  - Example: Past 7 days → `dt_date BETWEEN date(current_date, '-6 days') AND date(current_date)`
- **"since [time point]"**: From specified time point **to now**
  - Use `>=` operator, no end date needed
  - Example: Since last week → `dt_date >= date(current_date, 'start of week', '-1 week')`
  - **Note**: Different from "in [period]" which uses `BETWEEN` or `=` to limit to period range

**Cumulative Value Queries**: When querying cumulative values (wtd/mtd/qtd/ytd), if measure uses time modifiers, WHERE clause must use single date filter (e.g., `dt_date = date(current_date)`). Modifier automatically calculates cumulative value - no date range needed.

**Position Requirement**: Time filter conditions should be placed last in WHERE clause

### Time Calculation Functions

Use SQLite-style date functions:
```sql
date(expression, modifier_string, ...)
```

Supported modifiers:
- `NNN day(s)/week(s)/month(s)/quarter(s)/year(s)`: Add/subtract periods (negative N for past)
- `start of week/month/quarter/year`: First day of period (week starts Monday)
- `weekday N`: Nth day of week (1=Monday, 7=Sunday)

### Time Handling Prohibitions

- Prohibit mixing different time granularity fields
- Prohibit non-first-day format time filters
- Prohibit time modifier and GROUP BY granularity mismatch
- Prohibit meaningless average calculation granularity
- Prohibit mixing period-to-date modifiers with full-period GROUP BY

---

## Special Functions

- **String Matching**: INSTR function
- **Date Calculation**: `date(current_date, modifier, ...)`

---

## Query Pattern Specifications

- **TOP N Queries**: `ORDER BY ... LIMIT`
- **Extreme Value Queries**: `ORDER BY ... LIMIT 1`
- **ORDER BY Requirement**: When using ORDER BY, must explicitly specify ASC/DESC
- **Prohibit Unnecessary ORDER BY**: When query doesn't explicitly require sorting, don't add ORDER BY unless semantically necessary (like TOP N queries)

---

## Parameterization

Use `${}` placeholders for user-provided parameter values:
- **Numeric parameters**: Direct use (e.g., `LIMIT ${number}`, `> ${percentage} * 1.0 / 100`)
- **String parameters**: Wrap in single quotes (e.g., `product = '${name}'`)
- **Date modifiers**: Raw string (e.g., `'-${n} month'`)

---

## Special Scenarios

- **Moving Calculations**: Use `BETWEEN date(...) AND date(...)`
  - Past N days: Include current day
  - Past N months/quarters/years: Usually exclude current period

---

## Aggregation Query Specific Rules

### Measure Reference Rules

- **Basic Form**: Directly use `AGG(measure_name)`
- **Naming Conventions**:
  - Average measures: Add prefix/suffix (e.g., `annual_avg_revenue`, `avg_column_per_invoice`)
  - Extreme value measures: Add `min_`/`max_` prefix
  - Count measures: Add `number_of_` prefix
- **Modifier Usage**:
  - Allow `AGG(measure_name, modifier)` form
  - Prohibit `AGG({measure_name: modifier})` form
- **+f Modifier Syntax (Important)**:
  - When using `+f` modifier in DSQL AGG function, condition expression **must be written directly without quotes**
  - Correct: `AGG(sales, '+f', dt_date = date(current_date, '-1 day'))`
  - Wrong: `AGG(sales, '+f', 'dt_date = date(current_date, ''-1 day'')')` ❌
  - Note: This differs from DML definitions where `{+f: 'condition'}` format (with quotes) is used

### Measure Selection Principles

1. **Priority Order**:
   - First priority: Use measures already existing in `<base_model>`, `<model>`, or to be defined in ```dml```
   - Second priority: Use generic measures (e.g., `number_of_transactions`) combined with WHERE conditions or simple modifiers
   - If existing measure semantics match query requirements (even if name doesn't exactly match), use existing measure - prohibit creating new measure

2. **New Measure Creation Conditions**:
   - Only create new measure in ```dml``` when model truly lacks semantically matching measure AND cannot achieve requirement using existing measure with WHERE clause
   - **Prohibit creating overly specific, hard-to-reuse measures** (e.g., dedicated measures for specific customer/product)
     - Correct approach: Create generic measure, use WHERE clause in DSQL for filtering, not +f modifier for overly specific measure
   - **Checklist before creating new measure**:
     1. Does this measure represent a new business concept (not just dimension filtering)?
     2. Does this measure's calculation logic fundamentally differ from existing measures (not just different filter conditions)?
     3. Can this measure be achieved in DSQL via WHERE/HAVING with GROUP BY?
     - If answer to #3 is "yes", don't create new measure - use existing measure with GROUP BY in DSQL
   - Base new measures on existing measures rather than copying and modifying existing measure content, to maintain clear, concise, and separated concepts

3. **Complex Expression Handling**: If attempted measure definition involves complex expressions, must use datoml expressions to define in ```dml``` and reference directly in DSQL; prohibit complex expression calculations in DSQL

4. **Measure-Dimension Separation Principle**:
   - **Measures are business metrics**: Represent "what to calculate" (e.g., sales, outstanding amount), should remain generic
   - **Dimensions are analysis angles**: Represent "how to group" (e.g., customer, vendor, time), achieved via GROUP BY
   - **Prohibit creating dedicated measures for dimension combinations**: When query needs grouping by multiple dimensions, use generic measure with GROUP BY, not create new measure for each dimension combination
   - **Judgment Criteria**: If new measure only adds dimension filtering to existing measure (e.g., "customer is not null"), don't create new measure - use GROUP BY with that dimension field in DSQL

5. **Special Scenarios**:
   - For existence check scenarios, prohibit using aggregation measures (like `number_of_xxx`) - must return specific records to directly verify existence
   - When query involves growth rate/percentage calculations, prefer pop/yoy modifiers over manual growth rate calculations

### Output Field Rules

- SELECT fields should be determined by query requirements, only include fields directly relevant to query results, avoid unnecessary fields
- When query requires filtering entities (like customers, products) by measure values, even if not explicitly requesting to show measure values, SELECT clause must include both entity fields and corresponding measures (using AGG function)

### Time Modifier Handling

- When using modifiers, must include corresponding time granularity field

### Conditional Filtering Rules

1. **Avoid Duplicate Filtering**:
   - When measure name implies or definition includes required filter condition (e.g., `transaction_type = 'invoice'`), prohibit repeating in WHERE clause
   - Filters that can be added in WHERE clause don't need `+f` modifier when referencing measure

2. **Allow Extended Filtering**:
   - Can add other conditions in WHERE clause to further narrow result range
   - New conditions must form logical complement relationship with measure's original filter conditions

3. **String Matching Rules**:
   - Default to exact equality, except when query or `<model>` or `<base_model>` or `<preference>` explicitly requires fuzzy matching

4. **Measure Value Filtering**: WHERE cannot reference measures; when filtering by measure values must use HAVING

### Special Query Types

1. **Multi-Period Comparison**:
   - When query requires calculating growth rate/percentage or comparing growth magnitude, **prefer pop/mom/yoy modifiers** to directly get growth rate (decimal form)
   - When query requires showing both current and previous period values, use pp modifier to get previous period value
   - Only filter base period

2. **Period-to-Date Handling**:
   - **Method 1 (Preferred)**: Directly filter current period (prohibit cross-period comparison), e.g., `dt_week = date(current_date, 'start of week')`
   - **Method 2**: Use wtd/mtd/qtd/ytd modifiers, filter current day (allows cross-period comparison), and include `dt_date` field in GROUP BY

3. **Period Opening/Closing Values**:
   - Use pov/pcv modifiers

### Grouping Aggregation Fields

- **Time Modifier and GROUP BY Mandatory Requirement**: Whenever using time modifiers to reference any measure (like pop, pp, ytd, yoy), even without grouping statistics, must include corresponding time granularity field in GROUP BY
  - Example: When using `AGG(revenue, 'mom')`, GROUP BY must include `dt_month`
  - Example: When using `AGG(sales, 'ytd')`, GROUP BY must include `dt_date`
- Fields in GROUP BY that also appear in SELECT must appear in same order as SELECT
- **GROUP BY Essence**: GROUP BY groups data by dimensions; each group automatically includes measure values under that dimension - no need to create dedicated measures for each dimension combination
- **Multi-Dimension Query Handling**: When query involves multiple dimensions (like "A or B", "both A and B"), include these dimension fields in GROUP BY, use generic measures with HAVING filtering

---

## Detail Query Specific Rules

### Basic Requirements

- Must query from MODEL.dataset_name
- Prohibit using AGG function
- Prohibit using GROUP BY

### Output Field Rules

1. **Specific Field Queries**: When user queries only specific single field information (**except existence checks**), SELECT clause must contain only these specific fields, prohibit adding unrelated fields, and use `DISTINCT` keyword to remove duplicate records

2. **Core Entity Required**: When query involves specific products, customers, vendors, or other core entities, **must include corresponding entity fields in SELECT clause** (like `product`, `customer`), even if not explicitly requested, to ensure result relevance

3. **Other Cases**: SELECT output should only include key fields (dt_date, transaction_id, amount, etc.), example: `SELECT dt_date, transaction_id, amount...`, prohibit adding extra fields

### Period-to-Date Handling

- Period-to-date (MTD/QTD/YTD) only needs to filter corresponding time granularity field (like `dt_month`) equals current period start date
- **Absolutely prohibit** adding any `dt_date` related conditions

---

## Best Practices Summary

1. ✅ **Use single-line DSQL** - No line breaks or comments
2. ✅ **Choose correct query type** - Detail vs Aggregation based on priority rules
3. ✅ **Use AGG() for all measures** in aggregation queries
4. ✅ **Avoid redundant aliases** - AGG auto-aliases
5. ✅ **Minimize new definitions** - Reuse existing fields/measures
6. ✅ **Separate measures from dimensions** - Use GROUP BY for dimension combinations
7. ✅ **Use time modifiers** - Prefer pop/yoy over manual calculations
8. ✅ **Follow time field conventions** - Use first-day format
9. ✅ **Place time filters last** in WHERE clause
10. ✅ **Check column descriptions** - Follow "internal use only" instructions


