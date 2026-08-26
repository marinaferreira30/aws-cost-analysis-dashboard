# 1. AWS Cost Analysis Dashboard
SQL + Tableau project to analyze AWS cloud costs and provide dashboards for visibility and optimization.

**Interactive Dashboard:**  
👉 [View on Tableau Public](https://public.tableau.com/views/Book153211101copy2/Dashboard2?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

## 2. Summary
Using SQL and tableau I analyzed cost and usage report (CUR) data from an anonymized AWS environment. The intention was to analyze key spending trends over time by account, service, and usage type to identify areas of opportunity for optimization and cost reduction. I used SQL to quickly analyze these trends adhoc and build a self-service tableau dashboard which would enable users to answer billing questions themselves, without needing to ask me to manipulate data manually. In the future the dashboard would be section accessed via dynamic user functions as a data source filter to ensure that users only see their business' costs; leaving this unfiltered may cause privacy concerns due to sensitive nature of financial data.

## 3. Methodology
To explore AWS cost and usage data, I first generically queried 'select *' to familiarize myself with the schema. I spent time analyzing the rows to identify that the CUR comes in hourly increments, with an individual line item per account, service, region, and usage type. 

I then did some preliminary big picture analysis by querying for aggregating by month and service, to see which cloud products were the primary spend drivers. I did a similar query by account for the same reason. Lastly I utilized a window function to identify the primary month over month (MoM) deltas. The combination of these results grounded my understanding of the state of the cloud deployment and helped inform which visuals would be helpful in a dashboard.

As a final step I turned to tableau to build out a self-service dashboard. While querying of data is of course helpful for me to understand the state of cloud spend and answer adhoc questions, it is more sustainable and scalable to build a dashboard where product, finance, and engineering can go at their leisure to answer their own questions. In a real scenario I would further utilize user filters to ensure row level row level security given the sensitive nature of financial data. 


## 4. SQL Analysis
### 4.1 Monthly Spend Analysis

```sql
SELECT 
    TO_CHAR(date, 'YYYY-MM') AS month,
    SUM(cost) AS total_spend
FROM aws_cloud_costs
GROUP BY TO_CHAR(date, 'YYYY-MM')
ORDER BY month;
```
Result:
| month   | total_spend |
|---------|-------------|
| 2024-01 | 79824.68    |
| 2024-02 | 95829.15    |
| 2024-03 | 123932.70   |


### 4.2 Monthly Service-Level-Analysis

```sql
SELECT 
  to_char(date_trunc('month', date), 'YYYY-MM') AS month,
  SUM(CASE WHEN service = 'EC2'      THEN cost ELSE 0 END) AS ec2,
  SUM(CASE WHEN service = 'S3'       THEN cost ELSE 0 END) AS s3,
  SUM(CASE WHEN service = 'DynamoDB' THEN cost ELSE 0 END) AS dynamodb,
  SUM(CASE WHEN service = 'Lambda'   THEN cost ELSE 0 END) AS lambda,
  SUM(cost) AS total_spend
FROM aws_cloud_costs
GROUP BY month
ORDER BY month;

```
Result:

| month   | ec2      | s3       | dynamodb | lambda  | total_spend |
| ------- | -------- | -------- | -------- | ------- | ----------- |
| 2024-01 | 48184.53 | 19846.17 | 7830.26  | 3963.72 | 79824.68    |
| 2024-02 | 57523.05 | 23904.87 | 9608.75  | 4792.48 | 95829.15    |
| 2024-03 | 74044.63 | 31145.56 | 12519.32 | 6223.19 | 123932.70   |


### 4.3 MoM Service Deltas

```sql
WITH monthly AS (
  SELECT
    date_trunc('month', date) AS month,
    SUM(CASE WHEN service = 'EC2'      THEN cost ELSE 0 END) AS ec2,
    SUM(CASE WHEN service = 'S3'       THEN cost ELSE 0 END) AS s3,
    SUM(CASE WHEN service = 'DynamoDB' THEN cost ELSE 0 END) AS dynamodb,
    SUM(CASE WHEN service = 'Lambda'   THEN cost ELSE 0 END) AS lambda,
    SUM(cost) AS total_spend
  FROM aws_cloud_costs
  GROUP BY 1
)
SELECT
  to_char(month, 'YYYY-MM') AS month,
  (ec2        - LAG(ec2)        OVER (ORDER BY month)) AS ec2,
  (s3         - LAG(s3)         OVER (ORDER BY month)) AS s3,
  (dynamodb   - LAG(dynamodb)   OVER (ORDER BY month)) AS dynamodb,
  (lambda     - LAG(lambda)     OVER (ORDER BY month)) AS lambda,
  (total_spend- LAG(total_spend)OVER (ORDER BY month)) AS total_spend
FROM monthly
ORDER BY month;

```
Result:

| month   | ec2      | s3      | dynamodb | lambda  | total_spend |
| ------- | -------- | ------- | -------- | ------- | ----------- |
| 2024-01 | null     | null    | null     | null    | null        |
| 2024-02 | 9338.52  | 4058.70 | 1778.49  | 828.76  | 16004.47    |
| 2024-03 | 16521.58 | 7240.69 | 2910.57  | 1430.71 | 28103.55    |


## 5. Tableau Dashboard

I created an interactive dashboard in Tableau to visualize data based on service, usage type, region, and linked account.
Through the dashboard, you can: 
- View KPIs that show both the total cost for each month and the overall cumulative cost.
- Explore an interactive bar chart that allows you to switch between different views, including service, usage type, linked account, and regions.
- Acess an interactive donut chart that displays the percentage of total cost for each category, corresponding to the view of the bar chart.
- Analize a summary table that presents monthly spending by account, including total costs for each month and total costs per account.

[View Interactive Dashboard on Tableau Public](https://public.tableau.com/views/Book153211101copy2/Dashboard2?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

## 6. Analysis
EC2 spend dominated costs, which is unsurprising as this tends to be the case in most cloud deployments. It was interesting to see, though, that this spend (along with S3 requests) would drop over the weekend. This suggests this business is proportional to user traffic and thus may be tied to the financial week and would expect that the hourly spend is tied to market hours.

While total spend rose MoM, Account 1 declined over this period. This could suggest a dev account declining as production ramps, a product/feature with declining usage, or perhaps a migration to more efficient service. The total costs still rose over time though, indicating other accounts outweighed account 1's reduction.

Further, us-east-1 began as by far the largest spend, but over time us-west-2 grew as well, suggesting intended expansion in other region. Typically this will coorespond to intended business expansion to other regions to be closer to users in an attempt to lower latency and data transfer costs.
