# Compilation Error Troubleshooting

Common DML/Model Query SQL compilation errors and how to fix them.

## Unknown Column Errors

### Error: "Unknown column: X"

**Cause**: The column `X` is referenced in the query but not defined in the model.

**Solution**: Add a column-macro in `model.dml`:

```yaml
column-macros:
    transactions:
        X: actual_column_name
```

**Example**:
```
Error: Unknown column: customer
Fix: Add column-macro mapping customer to customer_name
```

```yaml
column-macros:
    transactions:
        customer: customer_name
```

## Unknown Measure Errors

### Error: "Unknown measure: Y"

**Cause**: The measure `Y` is referenced but not defined in the model.

**Solution**: Add measure definition in `model.dml`:

```yaml
Y:
    type: measure
    dataset: transactions
    formula: SUM(column_name)
```

**Example**:
```
Error: Unknown measure: sales
Fix: Add sales measure definition
```

```yaml
sales:
    type: measure
    description: Sales revenue from income accounts
    formula: |-
        {credit: {+f: {account_type: ['Income', 'Other Income']}}}
```

## Ambiguous Reference Errors

### Error: "Ambiguous reference to column: Z"

**Cause**: Column `Z` exists in multiple datasets without qualification.

**Solution**: Use fully qualified names:

```yaml
column-macros:
    transactions:
        city: '{DATASET_CUSTOMER}.city'
        account_type: '{DATASET_CHART_OF_ACCOUNTS}.account_type'
```

## Type Mismatch Errors

### Error: "Type mismatch: expected number, got string"

**Cause**: Attempting arithmetic operations on string columns.

**Solution**: Ensure column types are correct or add type conversion:

```yaml
column-macros:
    transactions:
        numeric_id: CAST(transaction_id AS INTEGER)
```

## Missing Dataset Errors

### Error: "Unknown dataset: DATASET_X"

**Cause**: Referenced dataset not defined in base_model.dml.

**Solution**: Regenerate base_model.dml using `list-models` with the correct data files.

## Circular Reference Errors

### Error: "Circular reference detected"

**Cause**: Measure or column-macro references itself directly or indirectly.

**Solution**: Break the circular dependency:

**Bad**:
```yaml
A:
    type: measure
    formula: {B}

B:
    type: measure
    formula: {A}  # Circular!
```

**Good**:
```yaml
A:
    type: measure
    formula: SUM(column_a)

B:
    type: measure
    formula: {A} + SUM(column_b)
```

## Syntax Errors

### Error: "Invalid YAML syntax"

**Cause**: Malformed YAML in model.dml.

**Common Issues**:
1. Incorrect indentation (use spaces, not tabs)
2. Missing colons after keys
3. Unquoted strings with special characters

**Solution**: Validate YAML syntax:

```yaml
# Bad
measure_name
    type: measure

# Good
measure_name:
    type: measure
```

### Error: "Invalid formula syntax"

**Cause**: SQL syntax error in formula.

**Common Issues**:
1. Missing quotes around string literals
2. Incorrect function names
3. Mismatched parentheses

**Solution**: Check SQL syntax:

```yaml
# Bad
formula: SUM(amount WHERE customer = John)

# Good
formula: SUM(amount WHERE customer = 'John')
```

## Filter Syntax Errors

### Error: "Invalid filter expression"

**Cause**: Incorrect inline filter syntax.

**Solution**: Use proper filter syntax:

```yaml
# Bad
formula: {sales: {filter: account_type = 'Income'}}

# Good
formula: {sales: {+f: {account_type: 'Income'}}}

# Also Good (for complex conditions)
formula: {sales: {+f: "instr(account, 'Office') > 0"}}
```

## Date Function Errors

### Error: "Invalid date function"

**Cause**: Incorrect date function syntax.

**Solution**: Use SQLite date functions:

```yaml
# Bad
formula: date(current_date, 'start month')

# Good
formula: date(current_date, 'start of month')

# Common date modifiers:
# 'start of year', 'start of month', 'start of quarter', 'start of week'
# '+1 day', '-1 month', '+3 months', '-1 year'
```

## Aggregation Errors

### Error: "Cannot use aggregation in column-macro"

**Cause**: Attempting to use AGG() in column-macro definition.

**Solution**: Use measures for aggregations, column-macros for row-level calculations:

```yaml
# Bad (column-macro)
column-macros:
    transactions:
        total_sales: AGG(amount)

# Good (measure)
total_sales:
    type: measure
    dataset: transactions
    formula: SUM(amount)
```

## Common Fixes Checklist

When you encounter a compilation error:

1. **Check spelling** - Ensure column/measure names match exactly
2. **Check case sensitivity** - Column names are case-sensitive
3. **Check indentation** - YAML requires consistent indentation (2 or 4 spaces)
4. **Check quotes** - String literals need single quotes in SQL formulas
5. **Check dataset reference** - Use `{DATASET_NAME}.column` for cross-dataset refs
6. **Check base_model.dml** - Ensure it's up-to-date with your data files
7. **Use compile tool** - Test changes with `compile` tool before `query`

## Debugging Strategy

1. **Start simple** - Test with basic query first
2. **Add complexity gradually** - Add filters/measures one at a time
3. **Use compile tool** - Validate model changes before executing queries
4. **Check error messages** - Read the full error message for clues
5. **Review examples** - Look at working examples in examples.yaml
6. **Regenerate base_model** - If structure changed, regenerate from data files

## Prevention Best Practices

1. **Use descriptive names** - Clear names reduce confusion
2. **Add descriptions** - Document complex formulas
3. **Test incrementally** - Test each change before adding more
4. **Follow naming conventions** - Consistent naming reduces errors
5. **Keep model.dml organized** - Group related definitions together
6. **Version control** - Track changes to model files
7. **Use examples as templates** - Copy patterns from working examples

