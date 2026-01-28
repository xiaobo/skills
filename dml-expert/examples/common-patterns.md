# Common Query Patterns

Real-world examples from production data analysis workflows.

## Time-based Queries

### Current Period

```sql
-- This month
SELECT AGG(sales) FROM MODEL WHERE dt_month = date(current_date, 'start of month')

-- This quarter
SELECT AGG(expense) FROM MODEL WHERE dt_quarter = date(current_date, 'start of quarter')

-- This year
SELECT AGG(revenue) FROM MODEL WHERE dt_year = date(current_date, 'start of year')

-- This week
SELECT dt_date, transaction_id, raw_amount 
FROM MODEL.transactions 
WHERE dt_week = date(current_date, 'start of week')
```

### Period-to-Date

```sql
-- Year to date
SELECT AGG(sales) FROM MODEL 
WHERE dt_date BETWEEN date(current_date, 'start of year') AND date(current_date)

-- Quarter to date
SELECT AGG(sales) FROM MODEL 
WHERE dt_date BETWEEN date(current_date, 'start of quarter') AND date(current_date)

-- Month to date
SELECT dt_date, transaction_id, raw_amount 
FROM MODEL.transactions 
WHERE dt_month = date(current_date, 'start of month')
```

### Historical Periods

```sql
-- Last month
SELECT AGG(sales) FROM MODEL 
WHERE dt_month = date(current_date, 'start of month', '-1 month')

-- Last quarter
SELECT AGG(sales) FROM MODEL 
WHERE dt_quarter = date(current_date, 'start of quarter', '-1 quarter')

-- Last 12 months
SELECT AGG(expense) FROM MODEL 
WHERE dt_month BETWEEN date(current_date, 'start of month', '-12 months') 
                   AND date(current_date, 'start of month', '-1 month')

-- Last 7 days
SELECT AGG(outgoing_payment) FROM MODEL 
WHERE dt_date BETWEEN date(current_date, '-6 days') AND date(current_date)
```

### Specific Months/Quarters

```sql
-- August this year
SELECT AGG(expense) FROM MODEL 
WHERE dt_month = date(current_date, 'start of year', '+7 months')

-- Q2 this year
SELECT AGG(expense) FROM MODEL 
WHERE dt_quarter = date(current_date, 'start of year', '+1 quarter')

-- May last year
SELECT AGG(outgoing) FROM MODEL 
WHERE dt_month = date(current_date, 'start of year', '-1 year', '+4 months')
```

## Aggregation Patterns

### Simple Aggregations

```sql
-- Total sales
SELECT AGG(sales) FROM MODEL

-- Sales by customer
SELECT customer, AGG(sales) FROM MODEL GROUP BY customer

-- Sales by product and month
SELECT product, dt_month, AGG(sales) FROM MODEL GROUP BY product, dt_month
```

### Filtered Aggregations

```sql
-- Sales for specific customer
SELECT AGG(sales) FROM MODEL WHERE customer = 'John Doe'

-- Expense on specific account
SELECT AGG(expense) FROM MODEL WHERE instr(account, 'Office Supplies') > 0

-- Transactions with specific payment method
SELECT AGG(number_of_transactions) FROM MODEL WHERE credit_card = 'yes'
```

### Top N Queries

```sql
-- Top 5 customers by sales
SELECT customer, AGG(sales) FROM MODEL 
GROUP BY customer 
ORDER BY AGG(sales) DESC 
LIMIT 5

-- Largest expense this year
SELECT dt_date, transaction_id, account, raw_amount 
FROM MODEL.transactions 
WHERE dt_year = date(current_date, 'start of year') 
  AND account_type IN ('Expense', 'Other Expense') 
ORDER BY raw_amount DESC 
LIMIT 1
```

### HAVING Clause

```sql
-- Customers with outstanding balance
SELECT customer, AGG(outstanding) FROM MODEL 
GROUP BY customer 
HAVING AGG(outstanding) > 0

-- Products selling less than last month
SELECT product, AGG(sales) as this_month, AGG(sales, 'pp') as last_month 
FROM MODEL 
WHERE dt_month = date(current_date, 'start of month') 
GROUP BY product 
HAVING AGG(sales) < AGG(sales, 'pp')
```

## Comparison Queries

### Period-over-Period

```sql
-- This month vs last month
SELECT AGG(sales) as this_month, AGG(sales, 'pp') as last_month 
FROM MODEL 
WHERE dt_month = date(current_date, 'start of month')

-- Year to date vs last year
SELECT AGG(sales, 'ytd') as this_ytd, AGG(sales, 'ytd', 'ya') as last_ytd 
FROM MODEL 
WHERE dt_date = date(current_date) 
GROUP BY dt_date
```

### Growth Rate

```sql
-- Revenue growth by account
SELECT account, AGG(revenue) as current, AGG(revenue, 'pp') as previous 
FROM MODEL 
WHERE dt_month = date(current_date, 'start of month') 
GROUP BY account 
HAVING AGG(revenue, 'pop') > 0.10  -- 10% growth
```

## Entity-specific Queries

### Customer Queries

```sql
-- Customer's total sales
SELECT customer, AGG(sales) FROM MODEL 
WHERE customer = 'Jane Smith' 
GROUP BY customer

-- Customer's payment history
SELECT dt_date, transaction_id, transaction_type, raw_amount 
FROM MODEL.transactions 
WHERE transaction_type = 'payment' AND customer = 'Jane Smith' 
ORDER BY dt_date DESC

-- Customers with AR > 90 days
SELECT customer, AGG(outstanding) FROM MODEL 
WHERE due_date < date(current_date, '-90 days') 
GROUP BY customer 
HAVING AGG(outstanding) > 0
```

### Product Queries

```sql
-- Product sales by month
SELECT product, dt_month, AGG(sales) FROM MODEL 
GROUP BY product, dt_month

-- Products never sold
SELECT product FROM MODEL 
GROUP BY product 
HAVING AGG(all_product_sales) = 0

-- Top selling products
SELECT product, AGG(sales) FROM MODEL 
GROUP BY product 
ORDER BY AGG(sales) DESC 
LIMIT 10
```

## Transaction Detail Queries

### Finding Specific Transactions

```sql
-- Most recent invoice for customer
SELECT dt_date, transaction_id, transaction_type, raw_amount 
FROM MODEL.transactions 
WHERE transaction_type = 'invoice' AND customer = 'John Doe' 
ORDER BY dt_date DESC 
LIMIT 1

-- All transactions for a trading partner
SELECT dt_date, transaction_id, customer, vendor, raw_amount 
FROM MODEL.transactions 
WHERE customer = 'ABC Corp' OR vendor = 'ABC Corp'

-- Transactions in a specific period
SELECT dt_date, transaction_id, raw_amount 
FROM MODEL.transactions 
WHERE dt_month = date(current_date, 'start of year', '+7 months')
```

## Fuzzy Matching

```sql
-- Search by account name (partial match)
SELECT AGG(expense) FROM MODEL 
WHERE instr(account, 'Food & drink') > 0

-- Search by product name
SELECT AGG(sales) FROM MODEL 
WHERE instr(product, 'Widget') > 0
```

## Financial Queries

### Outstanding/Overdue

```sql
-- Total outstanding (all unpaid)
SELECT AGG(outstanding) FROM MODEL WHERE customer = 'John Doe'

-- Balance due (overdue only)
SELECT AGG(balance_due) FROM MODEL WHERE customer = 'John Doe'

-- Unpaid invoices count
SELECT AGG(number_of_unpaid_invoices) FROM MODEL WHERE customer = 'John Doe'

-- Past due invoices since specific date
SELECT AGG(number_of_invoices) FROM MODEL 
WHERE due_date >= date(current_date, '-90 days') 
  AND open_balance > 0
```

### Payment Queries

```sql
-- Incoming payments (from customers)
SELECT AGG(incoming_payment) FROM MODEL 
WHERE customer = 'John Doe' 
  AND dt_month = date(current_date, 'start of month')

-- Outgoing payments (to vendors)
SELECT AGG(outgoing_payment) FROM MODEL 
WHERE vendor = 'Supplier Inc' 
  AND dt_quarter = date(current_date, 'start of quarter')
```

## Advanced Patterns

### Conditional Counts

```sql
-- Count of products with sales > threshold
SELECT DISTINCTCOUNTIF(product, AGG(sales, '+d', product) > 1000) as prod_cnt 
FROM MODEL
```

### Invoice Analysis

```sql
-- Invoice statistics
SELECT AGG(min_invoice_quantity) as min_qty, 
       AGG(avg_invoice_quantity) as avg_qty, 
       AGG(max_invoice_quantity) as max_qty 
FROM MODEL

-- Average invoice value
SELECT AGG(avg_invoice_value) FROM MODEL WHERE vendor = 'Supplier Inc'
```

