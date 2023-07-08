# data_bank
Step by step solution to 'Data with Danny's case study no. 4

Database schema can be found under https://8weeksqlchallenge.com/case-study-4/

![schema](https://github.com/xExuberantx/data_bank/assets/131042937/02398bcf-1cd6-4acd-bb78-0ab02faef4cd)

## A. Customer Nodes Exploration

### 1. How many unique nodes are there on the Data Bank system?
```
SELECT
    COUNT(DISTINCT node_id)
FROM data_bank.customer_nodes;
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/7ab7a59c-c186-45ac-a347-e9e6d6afde87)

### 2. What is the number of nodes per region?
```
SELECT
    region_name,
    COUNT(DISTINCT node_id) AS nodes
FROM data_bank.customer_nodes AS cn
LEFT JOIN data_bank.regions AS rg
USING(region_id)
GROUP BY region_name
ORDER BY nodes DESC;
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/5411a803-9bbd-4088-87bb-53a644a56ea8)


### 3. How many customers are allocated to each region?
```
SELECT
    region_name,
    COUNT(DISTINCT customer_id) AS customers
FROM data_bank.customer_nodes AS cn
LEFT JOIN data_bank.regions AS rg
USING(region_id)
GROUP BY region_name
ORDER BY customers DESC;
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/1d43a118-3b5a-4598-bf75-5bd054ca6f5d)

### 4. How many days on average are customers reallocated to a different node?
```
SELECT
    ROUND(AVG(end_date - start_date)) AS rel_time
FROM data_bank.customer_nodes
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/1628fee2-e85d-496d-b4a3-0dcba8eeb5fa)

### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
```
SELECT
    region_name,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY end_date - start_date) as median,
    PERCENTILE_DISC(0.8) WITHIN GROUP (ORDER BY end_date - start_date) as "80th",
    PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY end_date - start_date) as "95th"
FROM data_bank.customer_nodes
JOIN data_bank.regions 
USING(region_id)
GROUP BY region_name
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/9d087ba2-61bc-4c06-9986-921f351032d9)

## B. Customer Transactions

### 1. What is the unique count and total amount for each transaction type?
The 'unique count' seems a bit ambiguous as there are not transacion ids, but since it's supposed to be unique I counted uniqie customer_ids
```
SELECT
    txn_type,
    COUNT(DISTINCT customer_id),
    SUM(txn_amount) as total_amount
FROM data_bank.customer_transactions
GROUP BY txn_type
ORDER BY count DESC
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/7426085b-f6e7-4e15-8e5b-ab5ac30678e8)

### 2. What is the average total historical deposit counts and amounts for all customers?
```
WITH cte as (
    SELECT
        customer_id,
        COUNT(*),
        ROUND(AVG(txn_amount), 2) as avg_am_per_cust
    FROM data_bank.customer_transactions
    WHERE txn_type = 'deposit'
    GROUP BY customer_id)

SELECT
    ROUND(AVG(count)) as avg_cnt,
    ROUND(AVG(avg_am_per_cust), 2) as avg_amount
FROM cte
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/a3c84e73-1265-4d40-adc4-50ffa5018d49)

### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
Since the data is provided for ca. 4 months time we can simplify the query to just months
```
WITH deposits as (
    SELECT
        DATE_PART('month', txn_date) as month,
        customer_id,
        COUNT(customer_id) as depo_cnt
    FROM data_bank.customer_transactions
    WHERE txn_type = 'deposit'
    GROUP BY month, customer_id
    ORDER BY month DESC),
    purchases as (
    SELECT
        DATE_PART('month', txn_date) as month,
        customer_id,
        COUNT(customer_id) as purch_cnt
    FROM data_bank.customer_transactions
    WHERE txn_type = 'purchase'
    GROUP BY month, customer_id
    ORDER BY month DESC
    ),
    withdrawals as (
    SELECT
        DATE_PART('month', txn_date) as month,
        customer_id,
        COUNT(customer_id) withd_cnt
    FROM data_bank.customer_transactions
    WHERE txn_type = 'withdrawal'
    GROUP BY month, customer_id
    ORDER BY month DESC
    )
SELECT
    month,
    COUNT(*) as cnt
FROM deposits
FULL JOIN purchases
USING(month, customer_id)
FULL JOIN withdrawals
USING(month, customer_id)
WHERE depo_cnt > 1
  AND (purch_cnt = 1 OR withd_cnt = 1)
GROUP BY month
ORDER BY cnt DESC
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/a546e3d9-bfc5-43ea-b8fd-2d5fb28113a8)

### 4. What is the closing balance for each customer at the end of the month?
**Regarding each month individually**
```
SELECT
    customer_id,
    DATE_PART('month', txn_date) as month,
    SUM(
        CASE WHEN txn_type = 'deposit' THEN txn_amount
             ELSE -txn_amount END
    ) as balance
FROM data_bank.customer_transactions
GROUP BY customer_id, month
ORDER BY customer_id, month
LIMIT 15
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/6a85c8fc-ecad-425e-b5c9-51d6cff96b81)

**Regarding previous month's balance**
```
WITH balances as (
    SELECT
        customer_id,
        DATE_PART('month', txn_date) as month,
        SUM(
            CASE WHEN txn_type = 'deposit' THEN txn_amount
                ELSE -txn_amount END
        ) as balance
    FROM data_bank.customer_transactions
    GROUP BY customer_id, month
    ORDER BY customer_id, month)

SELECT
    customer_id,
    month,
    SUM(balance) OVER (PARTITION BY customer_id ORDER BY month RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
FROM balances
ORDER BY customer_id, month
```
![image](https://github.com/xExuberantx/data_bank/assets/131042937/af5aa2dd-b7f2-47de-b4e7-2b60a40acff4)


