# Measures Syntax Reference

Measures define aggregations and business metrics that can be used in Model Query SQL.

## Basic Syntax

```yaml
measure_name:
    type: measure
    dataset: dataset_name
    description: Business description
    formula: aggregation_expression
```

## Simple Aggregations

### COUNT

```yaml
number_of_transactions:
    type: measure
    dataset: transactions
    formula: count(distinct transaction_id)

number_of_products:
    type: measure
    dataset: transactions
    formula: count(distinct product)
```

### SUM

```yaml
amount:
    type: measure
    dataset: transactions
    formula: SUM({raw_amount})

transaction_quantity:
    type: measure
    dataset: transactions
    formula: SUM(quantity)
```

### MAX/MIN

```yaml
last_order_date:
    type: measure
    dataset: transactions
    formula: MAX(dt_date)
```

### AVG

```yaml
avg_invoice_quantity:
    type: measure
    formula: 'AVG({invoice_quantity: {+d: transaction_id}})'
```

## Filtered Measures

Use inline filters with `{+f: condition}`:

```yaml
expense:
    type: measure
    dataset: transactions
    formula: |-
        {debit: {+f: {account_type: ['Expense', 'Other Expense']}}}

sales:
    type: measure
    description: Sales revenue from income accounts
    formula: |-
        {credit: {+f: {account_type: ['Income', 'Other Income']}}}
```

## Measure References

Reference other measures in formulas:

```yaml
# Base measures
credit:
    type: measure
    dataset: transactions
    formula: SUM(credit)

debit:
    type: measure
    dataset: transactions
    formula: SUM(debit)

# Derived measure
profit:
    type: measure
    formula: |-
        {credit: {+f: {transaction_type: ['Income','Other Income','Expense','Other Expense']}}} - 
        {debit: {+f: {transaction_type: ['Income','Other Income','Expense','Other Expense']}}}
```

## Measure Aliases

Create aliases for existing measures:

```yaml
revenue: |-
    {sales}

income: |-
    {sales}

outgoing: |-
    {expense}
```

## Complex Formulas

### Conditional Aggregations

```yaml
incoming_payment:
    type: measure
    formula: |-
        SUMIF({payment: {+d: customer}}, {customer} is not null)

outgoing_payment:
    type: measure
    formula: |-
        SUMIF({payment: {+d: vendor}}, {vendor} is not null)
```

### Nested Aggregations

```yaml
max_amount_per_invoice:
    type: measure
    formula: |-
        MAX({sales: {+d: transaction_id}})

daily_avg_sales:
    type: measure
    formula: |-
        divide(sum({sales: {+d: dt_date}}), distinctcount({dt_date}))

weekly_avg_sales:
    type: measure
    formula: |-
        divide(sum({sales: {+d: dt_week}}), distinctcount({dt_week}))
```

### Window Functions

```yaml
product_rank_by_sales:
    type: measure
    formula: |-
        rank() over (partition by {product} order by {sales: {+d: product}} desc)
```

### ISNULL and IIF

```yaml
all_product_sales:
    type: measure
    description: Product sales including products with zero sales
    formula: |-
        SUM(isnull({sales: {+d: product}}, 0) * iif({DATASET_PRODUCT_SERVICE}.product is not null, 1, 0))

all_customer_sales:
    type: measure
    description: Customer sales including customers with zero sales
    formula: |-
        SUM(isnull({sales: {+d: customer}}, 0) * iif({DATASET_CUSTOMER}.customer is not null, 1, 0))
```

## Inline Filters and Dimensions

### Inline Filters `{+f: condition}`

```yaml
food_and_drink_expense:
    type: measure
    formula: |-
        {expense: {+f: "instr(account, 'Food & drink') > 0"}}

number_of_credit_card_transactions:
    type: measure
    formula: |-
        {number_of_transactions: {+f: {credit_card : 'yes'}}}

number_of_unpaid_invoices:
    type: measure
    formula: |-
        {number_of_invoices: {+f: 'open_balance > 0'}}
```

### Inline Dimensions `{+d: dimension}`

```yaml
avg_sales_per_trx:
    type: measure
    formula: |-
        divide(sum({sales: {+d: transaction_id}}), distinctcount({transaction_id}))
```

## Business Metrics Examples

### Financial Metrics

```yaml
account_receivable:
    type: measure
    formula: |-
        {amount: {+f: {account: 'Accounts Receivable (A/R)'}}}

outstanding:
    type: measure
    description: All unpaid amounts (including not yet due)
    formula: |-
        {open_balance}

balance_due:
    type: measure
    description: Overdue amounts only (past due_date)
    formula: |-
        {open_balance: {+f: 'due_date <= date(current_date)'}}
```

### Transaction Counts

```yaml
number_of_invoices:
    type: measure
    formula: |-
        {number_of_transactions: {+f: {transaction_type : 'invoice'}}}

number_of_checks:
    type: measure
    formula: |-
        {number_of_transactions: {+f: {transaction_type : 'check'}}}

number_of_received_checks:
    type: measure
    formula: |-
        {number_of_checks: {+f: 'customer is not null'}}
```

## Best Practices

1. **Add descriptions** to explain business meaning
2. **Use measure references** to build complex metrics from simple ones
3. **Avoid redundant filters** - don't repeat filters already in the measure definition
4. **Use inline filters** for conditional aggregations
5. **Document edge cases** (e.g., "includes zero values" or "excludes not yet due")
6. **Prefer business names** over technical column names

## Common Patterns

### Average Calculations

```yaml
avg_invoice_value:
    type: measure
    formula: |-
        divide({amount: {+f: {transaction_type: 'invoice'}}}, {number_of_invoices})
```

### Frequency Calculations

```yaml
avg_bill_frequency:
    type: measure
    formula: |-
        divide(sum({number_of_invoices: {+d: dt_date}}), datediff(day, max({dt_date}), min({dt_date})) + 1)
```

### Year-over-Year Metrics

```yaml
annual_avg_revenue:
    type: measure
    formula: |-
        divide(sum({revenue: {+d: dt_year}}), distinctcount({dt_year}))
```

