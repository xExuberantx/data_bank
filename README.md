# data_bank
Step by step solution to 'Data with Danny's case study no. 4

Database schema can be found under https://8weeksqlchallenge.com/case-study-4/

![schema](https://github.com/xExuberantx/data_bank/assets/131042937/02398bcf-1cd6-4acd-bb78-0ab02faef4cd)

```
-- 1. How many unique nodes are there on the Data Bank system?
SELECT
    COUNT(DISTINCT node_id)
FROM data_bank.customer_nodes;
```
count
5
