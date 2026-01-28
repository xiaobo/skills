# Business Logic Rules

Domain-specific rules for interpreting natural language queries in business contexts.

## Transaction Type Mappings

### Bill
- **"bill from vendor"** → `transaction_type = 'bill'`, trading partner is vendor
- **"bill to customer"** → `transaction_type = 'invoice'`, trading partner is customer
- **Context matters**: "bill" can mean invoice (我方发出) or bill (对方发出)

### Purchase
- **"we purchased from vendor"** → `transaction_type = 'purchase order'`, vendor field
- **"customer purchased from us"** → `transaction_type IN ('invoice', 'sales receipt')`, customer field, use `sales` measure
- **Key distinction**: Direction of purchase determines transaction type

### Payment
- **"payment from customer"** → `transaction_type = 'payment'`, customer field, use `incoming_payment`
- **"payment to vendor"** → `transaction_type = 'payment'`, vendor field, use `outgoing_payment`

### Invoice
- **"receiving invoice"** / **"invoice from"** → vendor is trading partner
- **"issuing invoice"** / **"invoice to"** → customer is trading partner

## Financial Concepts

### Earned / Income / Amount Earned
- Use `income` or `sales` measure (they're equivalent)
- Filter: `account_type IN ('Income', 'Other Income')`
- Trading partner: customer
- **Preference**: Use `income` measure when query mentions "earned" or "income"

### Expense
- Use `expense` measure
- Filter: `account_type IN ('Expense', 'Other Expense')`

### Outstanding
- **Definition**: All unpaid amounts (includes not yet due + overdue)
- Use `outstanding` measure
- **Time field**: `due_date` (not `dt_date`)
- **HAVING clause rules**:
  - **Filtering/comparison queries**: Add `HAVING AGG(outstanding) > 0` to exclude zero values
  - **Direct queries**: Don't add HAVING (return 0 if no outstanding)
- **"as of [date]" with "still outstanding"**: No time filter needed (it's a status query, not time range)

### Balance Due / Past Due / Overdue
- **Definition**: Only overdue amounts (past `due_date`)
- Use `balance_due` measure
- **Time field**: `due_date` (not `dt_date`)
- **Time filters**:
  - "since [date]" → `due_date >= [date]`
  - "in [period]" → `due_date BETWEEN [start] AND [end]`

### Unpaid / Gone Unpaid
- **Definition**: Invoices not yet paid
- **Time field**: `dt_date` (transaction date, not due date)
- **Distinction**: "unpaid" (based on transaction date) vs "past due" (based on due date)

### Open Credit
- Trading partner: customer
- Use `open_balance` measure

### AR (Accounts Receivable)
- Use `ar_in_days` column-macro for AR aging
- Note: AR is created with invoice, but invoice record itself doesn't have AR property

## Trading Partner Rules

### Customer vs Vendor
- **Customer**: Use `customer` field (not `customer_name`)
- **Vendor**: Use `vendor` field (not `vendor_name`)
- **Ambiguous**: When direction unclear, check both `customer` and `vendor` fields
- **Exception**: Only use `customer_name` or `vendor_name` if explicitly requested

### "Someone owes us"
- Trading partner: customer
- Use `outstanding` for all unpaid, `balance_due` for overdue only

## Field Usage Rules

### Account Field
- **Always use fuzzy matching**: `instr(account, 'search_term') > 0`
- **Never use exact match**: `account = 'value'` is incorrect

### Time Fields
- **dt_date**: Transaction date (use for "when did X happen")
- **due_date**: Payment due date (use for "past due" / "overdue")
- **fiscal_year**: Fiscal year starting in April (only use when explicitly mentioned)
- **dt_year**: Natural calendar year (default for year queries)

### Product/Service Fields
- **product**: Use for product-related queries
- **service**: Maps to `product_service_name`
- **equipment**: Maps to `product_service_name`
- **product_service_name**: Internal use only, use more specific fields

## Query Interpretation Rules

### "How often" / "Frequency"
- **Not total count**: User wants rate (per month, per year, etc.)
- Use frequency measures like `avg_bill_frequency`
- Example: "How often do we bill X?" → Use `avg_bill_frequency`, not `number_of_invoices`

### "Never sold" / "Never been sold"
- **Alone**: Completely unsold (sales = 0), use `all_product_sales = 0`
- **With threshold**: "never sold with value > X" means sales ≤ X, use `sales <= X`

### "Details of customer"
- Use `DATASET_CUSTOMER` dataset
- Query: `SELECT * FROM MODEL.DATASET_CUSTOMER WHERE customer = 'name'`

### "Recurring purchases"
- Filter for multiple occurrences: `HAVING AGG(number_of_transactions) > 1`
- Group by product to find repeated purchases

## Measure Usage Rules

### Don't Repeat Filters
- **Rule**: If measure definition includes filter, don't repeat in WHERE clause
- **Example**: `sales` already filters `account_type IN ('Income', 'Other Income')`
  - ❌ Bad: `SELECT AGG(sales) FROM MODEL WHERE account_type IN ('Income', 'Other Income')`
  - ✅ Good: `SELECT AGG(sales) FROM MODEL`

### Don't Add Extra Conditions to Defined Measures
- **Rule**: If measure has clear semantic definition, don't add contradictory filters
- **Example**: `outstanding` means "all unpaid, not checking if overdue"
  - ❌ Bad: `SELECT AGG(outstanding) FROM MODEL WHERE due_date < current_date`
  - ✅ Good: `SELECT AGG(outstanding) FROM MODEL` or use `balance_due` instead

### Use Existing Fields/Measures
- **Rule**: If model has field/measure matching business concept, use it directly
- **Example**: If `payment_status` column-macro exists
  - ❌ Bad: `WHERE open_balance > 0` (reimplementing logic)
  - ✅ Good: `WHERE payment_status = 'unpaid'`

## Time Period Interpretation

### "This [period]"
- **This month**: `dt_month = date(current_date, 'start of month')`
- **This quarter**: `dt_quarter = date(current_date, 'start of quarter')`
- **This year**: `dt_year = date(current_date, 'start of year')`
- **This week**: `dt_week = date(current_date, 'start of week')`

### "[Period] to date"
- **Year to date**: `dt_date BETWEEN date(current_date, 'start of year') AND date(current_date)`
- **Quarter to date**: `dt_date BETWEEN date(current_date, 'start of quarter') AND date(current_date)`
- **Month to date**: `dt_month = date(current_date, 'start of month')`

### "Last [period]"
- **Last month**: `dt_month = date(current_date, 'start of month', '-1 month')`
- **Last quarter**: `dt_quarter = date(current_date, 'start of quarter', '-1 quarter')`
- **Last year**: `dt_year = date(current_date, 'start of year', '-1 year')`

### "Last N [units]"
- **Last 7 days**: `dt_date BETWEEN date(current_date, '-6 days') AND date(current_date)`
- **Last 12 months**: `dt_month BETWEEN date(current_date, 'start of month', '-12 months') AND date(current_date, 'start of month', '-1 month')`

## Preference Application Rules

1. **Must apply preference mappings**: When preference defines term → field/condition mapping, must use it
2. **Distinguish similar concepts**: Carefully differentiate between similar business terms
3. **Avoid over-complexity**: Use existing fields/measures instead of rebuilding logic
4. **Check measure definitions**: Review formula before using to avoid redundant filters
5. **Context is key**: Same word can mean different things in different contexts

## Common Mistakes to Avoid

1. ❌ Using `dt_date` for "past due" queries (should use `due_date`)
2. ❌ Adding `transaction_type = 'invoice'` when using `number_of_invoices` (already filtered)
3. ❌ Using exact match on `account` field (should use `instr()`)
4. ❌ Confusing "outstanding" (all unpaid) with "balance_due" (overdue only)
5. ❌ Using `customer_name` instead of `customer` (unless explicitly requested)
6. ❌ Treating "frequency" as total count (should be rate per period)
7. ❌ Adding time filters to "as of [date]" status queries

