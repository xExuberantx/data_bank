# data_bank
Step by step solution to 'Data with Danny's case study no. 4

Database schema can be found under https://8weeksqlchallenge.com/case-study-4/

![schema](https://github.com/xExuberantx/data_bank/assets/131042937/02398bcf-1cd6-4acd-bb78-0ab02faef4cd)

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
