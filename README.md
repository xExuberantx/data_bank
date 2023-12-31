# data_bank
Step by step solution to 'Data with Danny's case study no. 4

Database schema can be found under https://8weeksqlchallenge.com/case-study-4/

![image](screens/schema.png)


## A. Customer Nodes Exploration

### 1. How many unique nodes are there on the Data Bank system?
```
SELECT
    COUNT(DISTINCT node_id)
FROM data_bank.customer_nodes;
```
![image](screens/1.png)

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
![image](screens/2.png)


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
![image](screens/3.png)

### 4. How many days on average are customers reallocated to a different node?
```
SELECT
    ROUND(AVG(end_date - start_date)) AS rel_time
FROM data_bank.customer_nodes
```
![image](screens/4.png)

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
![image](screens/5.png)

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
![image](screens/b1.png)

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
![image](screens/b2.png)

### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
Since the data is provided for ca. 4 months time we can simplify the query to just months

```
WITH txns as (
    SELECT
        DATE_PART('month', txn_date) as month,
        customer_id,
        SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) as deposits,
        SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) as purchases,
        SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) as withdrawals
    FROM data_bank.customer_transactions
    GROUP BY month, customer_id
    ORDER BY month, customer_id
)
SELECT
    month,
    COUNT(*) as cnt
FROM txns
WHERE deposits > 1 AND (purchases = 1 OR withdrawals = 1)
GROUP BY month
ORDER BY cnt DESC
```
![image](screens/b3.png)

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
![image](screens/b4.png)

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
    SUM(balance) OVER (PARTITION BY customer_id ORDER BY month RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as balance2
FROM balances
ORDER BY customer_id, month
```
![image](screens/b4b.png)

### 5. What is the percentage of customers who increase their closing balance by more than 5%?
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
    ORDER BY customer_id, month),

    balances2 as (
    SELECT
        customer_id,
        month,
        SUM(balance) OVER (PARTITION BY customer_id ORDER BY month RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as balance2
    FROM balances
    ORDER BY customer_id, month),

    increase as (
    SELECT
        customer_id,
        ((SELECT balance2 FROM balances2 as bal WHERE bal.customer_id=b.customer_id
                                                AND month = (SELECT MAX(month) FROM balances2 as bal2 WHERE bal2.customer_id=bal.customer_id)) -
        (SELECT balance2 FROM balances2 as bal WHERE bal.customer_id=b.customer_id
                                                AND month = (SELECT MIN(month) FROM balances2 as bal2 WHERE bal2.customer_id=bal.customer_id)))*100/
        (SELECT balance2 FROM balances2 as bal WHERE bal.customer_id=b.customer_id
                                                AND month = (SELECT MIN(month) FROM balances2 as bal2 WHERE bal2.customer_id=bal.customer_id)) as perc_incr
    FROM balances2 as b
    GROUP BY customer_id)

SELECT
    ROUND((SELECT COUNT(*) FROM increase WHERE perc_incr > 5)::NUMERIC/(SELECT COUNT(*) FROM increase)*100, 2) as cust_with_incr
```
![image](screens/b5.png)

## C. Data Allocation Challenge

### Running customer balance column that includes the impact each transaction
```
SELECT
    *,
    SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
        ELSE -txn_amount END) OVER (PARTITION BY customer_id ORDER BY txn_date RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as balance
FROM data_bank.customer_transactions
LIMIT 15
```
![image](screens/c_balance.PNG)

### Customer balance at the end of each month
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
    SUM(balance) OVER (PARTITION BY customer_id ORDER BY month RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as balance2
FROM balances
ORDER BY customer_id, month
LIMIT 15
```
![image](screens/c_bal_eom.PNG)

### Minimum, average and maximum values of the running balance for each customer
```
WITH balances as (
    SELECT
        *,
        SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
            ELSE -txn_amount END) OVER (PARTITION BY customer_id ORDER BY txn_date RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as balance
    FROM data_bank.customer_transactions)

SELECT
    customer_id,
    ROUND(MIN(balance), 2) as min,
    ROUND(AVG(balance), 2 ) as avg,
    ROUND(MAX(balance), 2) as max
FROM balances
GROUP BY customer_id
LIMIT 15
```
![image](screens/c_summary.PNG)

### Option 1: data is allocated based off the amount of money at the end of the previous month


