# Column Macros Syntax Reference

Column macros allow you to create aliases, computed fields, and cross-dataset references.

## Basic Syntax

```yaml
column-macros:
    dataset_name:
        macro_name: column_reference_or_formula
```

## Simple Aliases

Map a user-friendly name to an actual column:

```yaml
column-macros:
    transactions:
        account_name: account
        dt_date: transaction_date
        customer: customer_name
        vendor: vendor_name
        product: product_service_name
```

## Cross-Dataset References

Reference columns from other datasets using `{DATASET_NAME}.column`:

```yaml
column-macros:
    transactions:
        account_type: '{DATASET_CHART_OF_ACCOUNTS}.account_type'
        credit_card: '{DATASET_PAYMENT_METHOD}.credit_card'
        city: '{DATASET_CUSTOMER}.city'
```

**Note**: Use single quotes around the reference.

## Computed Fields with Formulas

### CASE Expressions

```yaml
column-macros:
    transactions:
        ar_in_days: case when {open_balance} > 0 then datediff(day, current_date, {dt_date}) else 0 end
        payment_status: case when {open_balance} > 0 then 'unpaid' else 'paid' end
```

### Date Functions

```yaml
column-macros:
    transactions:
        fiscal_year: 
            description: |-
                Fiscal year starting in April (use dt_year for natural year)
            formula: date({dt_date}, '-3 months', 'start of year', '+3 months')
```

### Date Difference

```yaml
column-macros:
    transactions:
        days_past_due: datediff(day, due_date, current_date)
```

## Multi-line Formulas

Use `|-` for multi-line formulas:

```yaml
column-macros:
    transactions:
        city: |-
            {DATASET_CUSTOMER}.city
```

## Column Descriptions

Add descriptions to document business logic:

```yaml
column-macros:
    transactions:
        fiscal_year: 
            description: |-
                Fiscal year starting in April
                Use dt_year for natural calendar year queries
            formula: date({dt_date}, '-3 months', 'start of year', '+3 months')
```

## Referencing Other Macros

You can reference other column macros using `{macro_name}`:

```yaml
column-macros:
    transactions:
        dt_date: transaction_date
        ar_in_days: case when {open_balance} > 0 then datediff(day, current_date, {dt_date}) else 0 end
```

## Multiple Datasets

Define macros for different datasets:

```yaml
column-macros:
    transactions:
        customer: customer_name
        product: product_service_name
    
    DATASET_PRODUCT_SERVICE:
        product: product_service_name
        flag: 1
    
    DATASET_CUSTOMER:
        customer: customer_name
```

## Best Practices

1. **Use descriptive names** that match business terminology
2. **Add descriptions** for complex formulas or business rules
3. **Reference base columns** from base_model.dml, not raw table columns
4. **Use cross-dataset references** to enrich data with lookup values
5. **Document time field semantics** (transaction date vs due date vs fiscal year)

## Common Patterns

### Fuzzy Match Guidance

For columns that require fuzzy matching, add description:

```yaml
column-macros:
    transactions:
        account:
            description: |-
                Use instr(account, 'search_term') > 0 for partial matching
                Do not use exact match (account = 'value')
```

### Internal Use Only

Mark columns that should not be used directly:

```yaml
columns:
    customer_name:
        description: |-
            internal use only
            use column customer instead of customer_name in query
```

## Examples from Real Projects

### Financial Transaction System

```yaml
column-macros:
    transactions:
        # Aliases
        account_name: account
        dt_date: transaction_date
        payment_method_type: payment_method
        
        # Cross-dataset lookups
        account_type: '{DATASET_CHART_OF_ACCOUNTS}.account_type'
        credit_card: '{DATASET_PAYMENT_METHOD}.credit_card'
        
        # Computed fields
        ar_in_days: case when {open_balance} > 0 then datediff(day, current_date, {dt_date}) else 0 end
        payment_status: case when {open_balance} > 0 then 'unpaid' else 'paid' end
        days_past_due: datediff(day, due_date, current_date)
        
        # Date transformations
        fiscal_year: date({dt_date}, '-3 months', 'start of year', '+3 months')
        
        # Business entity mappings
        customer: customer_name
        vendor: vendor_name
        product: product_service_name
        service: product_service_name
        equipment: product_service_name
```

