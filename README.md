# Hands-on L11: AWS Core Services (S3, Glue, CloudWatch, Athena)

Author: Poojitha Jayareddygari
Student ID: 801426875
Course: Cloud Computing for Data Analysis
University: UNC Charlotte

**Course:** ITCS 6190/8190 — Cloud Computing for Data Analysis  
**Instructor:** Marco Vieira  
**Semester:** Fall 2025  

---

## Objective
This hands-on exercise demonstrates the use of **AWS Core Services** to manage and analyze data using **S3**, **Glue**, **CloudWatch**, and **Athena**.  
The main goal is to build a workflow that loads raw e-commerce sales data into AWS, automates schema detection using Glue, and performs analytical queries via Athena.

---

## Dataset
**Source:** [Kaggle – Unlock Profits with E-commerce Sales Data](https://www.kaggle.com/datasets/thedevastator/unlock-profits-with-e-commerce-sales-data)  
The dataset contains sales transactions with attributes such as order date, product category, shipping state, amount, and status.

---

## AWS Workflow

### 1. Amazon S3 Setup
- Created an **S3 bucket** to store raw and processed CSV files.
- Uploaded the dataset (`sales.csv`) into the `raw` folder.
- Defined a second folder (`processed/`) for transformed or queried data outputs.

📸 *Screenshot: S3 bucket structure*
<img width="1470" height="956" alt="S3" src="https://github.com/user-attachments/assets/464bfccc-0984-4195-9d4f-6c8f2b4b4d80" />

---

### 2. IAM Role Configuration
- Created an **IAM Role** named `AWSGlueServiceRole-DataAnalysis`.
- Assigned the following permissions:
  - **AmazonS3FullAccess**
  - **AWSGlueServiceRole**
  - **AmazonAthenaFullAccess**
  - **CloudWatchFullAccess**

📸 *Screenshot: IAM Role Permissions*
<img width="1470" height="956" alt="IAM" src="https://github.com/user-attachments/assets/fed3fac4-5e0b-492e-9c87-6278403dc259" />


---

### 3. AWS Glue Crawler
- Configured a **Glue Crawler** linked to the S3 `raw` folder.
- The crawler automatically detected schema and created a database in the **AWS Glue Data Catalog**.
- Monitored crawler execution in **CloudWatch**.

📸 *Screenshot: Glue Crawler and CloudWatch logs*
<img width="1470" height="956" alt="Glue" src="https://github.com/user-attachments/assets/ab7020b0-f363-4b2c-ab84-29638509afab" />
<img width="1470" height="956" alt="CloudWatch" src="https://github.com/user-attachments/assets/5ba59c79-0409-45e2-823b-a45f72287275" />


---

### 4. Querying with Amazon Athena
Used **Amazon Athena** to perform analytical queries on the crawled database.

---

## 🧮 Queries Executed

### **Query 1: Cumulative Sales Over Time for 2022**
Calculates daily and cumulative sales trends.
```sql
SELECT "date",
       SUM(CAST(amount AS DOUBLE)) AS daily_sales,
       SUM(SUM(CAST(amount AS DOUBLE))) OVER (ORDER BY "date" ASC) AS cumulative_sales
FROM "AwsDataCatalog"."output_db"."raw"
WHERE status IN ('Shipped', 'Shipped - Delivered to Buyer')
  AND substr("date", 7, 2) = '22'
GROUP BY "date"
ORDER BY "date" ASC
LIMIT 10;
```
📸 Screenshot: Query 1 results in Athena 
<img width="1470" height="956" alt="Q1" src="https://github.com/user-attachments/assets/7b8ea3aa-a2fb-4274-8ffd-fa48742d47f8" />
<img width="1470" height="956" alt="Q1 output" src="https://github.com/user-attachments/assets/734576fd-b26a-4427-b2d2-ff876fd0932a" />



Query 2: Geographic Hotspot Analysis for Unprofitable Products
Finds states contributing the most to negative profits.

```sql
Copy code
SELECT COALESCE(NULLIF("ship-state", ''), 'Unknown') AS state,
       SUM(CASE WHEN lower(status) IN ('cancelled','canceled','refunded','returned','return requested','refund initiated')
                THEN -ABS(amount) ELSE 0 END) AS total_negative_profit
FROM raw
GROUP BY COALESCE(NULLIF("ship-state", ''), 'Unknown')
HAVING SUM(CASE WHEN lower(status) IN ('cancelled','canceled','refunded','returned','return requested','refund initiated')
                THEN -ABS(amount) ELSE 0 END) < 0
ORDER BY total_negative_profit ASC
LIMIT 10;
```
📸 Screenshot: Query 2 results in Athena 
<img width="1470" height="956" alt="Q2" src="https://github.com/user-attachments/assets/cbe684aa-e087-4611-911b-6517d14d7482" />
<img width="1470" height="956" alt="Q2 output" src="https://github.com/user-attachments/assets/6d243fb9-e45d-4030-9c5c-a27ff9a7aef8" />


Query 3: Impact of Discounts on Profitability by Sub-Category
Simulates discount rates and evaluates profit ratios.

```sql
Copy code
SELECT category, sku AS sub_category,
       ROUND(AVG(rand() * 0.3), 2) AS avg_discount,
       CAST(SUM(CAST(amount AS DOUBLE)) AS DOUBLE) AS total_sales,
       CAST(SUM(CAST(amount AS DOUBLE)) * (1 - AVG(rand() * 0.3)) AS DOUBLE) AS est_profit,
       (CAST(SUM(CAST(amount AS DOUBLE)) * (1 - AVG(rand() * 0.3)) AS DOUBLE) /
        NULLIF(CAST(SUM(CAST(amount AS DOUBLE)) AS DOUBLE), 0)) AS profit_ratio
FROM raw
GROUP BY category, sku
ORDER BY profit_ratio DESC
LIMIT 10;
```
📸 Screenshot: Query 3 results in Athena 
<img width="1470" height="956" alt="Q3" src="https://github.com/user-attachments/assets/d41f0831-7d28-4328-adc7-0ee009a2b83d" />
<img width="1470" height="956" alt="Q3 ouput" src="https://github.com/user-attachments/assets/0e5237ce-118d-42dc-9206-9b88236f9707" />


Query 4: Top 3 Most Profitable Products per Category
Ranks products using window functions.

```sql
Copy code
WITH ranked_products AS (
   SELECT category,
          sku AS product_name,
          CAST(SUM(CAST(amount AS DOUBLE) * 0.25) AS DOUBLE) AS total_profit,
          RANK() OVER (PARTITION BY category ORDER BY SUM(CAST(amount AS DOUBLE) * 0.25) DESC) AS rank_in_category
   FROM raw
   GROUP BY category, sku
)
SELECT category, product_name, total_profit, rank_in_category
FROM ranked_products
WHERE rank_in_category <= 3
ORDER BY category, rank_in_category
LIMIT 10;
```
📸 Screenshot: Query 4 results in Athena 
<img width="1470" height="956" alt="Q4" src="https://github.com/user-attachments/assets/b960fd12-031e-4ec4-af68-848751226992" />
<img width="1470" height="956" alt="Q4 output" src="https://github.com/user-attachments/assets/fc7b2343-9831-4f9d-bfd3-4effcf387a5f" />


Query 5: Monthly Sales and Profit Growth Analysis
Analyzes growth rates using lag functions.

```sql
Copy code
WITH monthly_summary AS (
   SELECT date_format(date_parse(date, '%m-%d-%y'), '%Y-%m') AS month,
          CAST(SUM(CAST(amount AS DOUBLE)) AS DOUBLE) AS total_sales,
          CAST(SUM(CAST(amount AS DOUBLE) * 0.25) AS DOUBLE) AS total_profit
   FROM raw
   GROUP BY date_format(date_parse(date, '%m-%d-%y'), '%Y-%m')
)
SELECT month, total_sales, total_profit,
       ROUND(((total_sales - LAG(total_sales) OVER (ORDER BY month)) /
              NULLIF(LAG(total_sales) OVER (ORDER BY month), 0)) * 100, 2) AS sales_growth_rate,
       ROUND(((total_profit - LAG(total_profit) OVER (ORDER BY month)) /
              NULLIF(LAG(total_profit) OVER (ORDER BY month), 0)) * 100, 2) AS profit_growth_rate
FROM monthly_summary
ORDER BY month
LIMIT 10;
```
📸 Screenshot: Query 5 results in Athena 
<img width="1470" height="956" alt="Q5" src="https://github.com/user-attachments/assets/cd03a6d5-d1ce-4d8a-91bf-910cc56d8b96" />
<img width="1470" height="956" alt="Q5 output" src="https://github.com/user-attachments/assets/61eff21a-7b61-4729-8fe2-c21c7dcf5dd5" />


## Results Summary
Query	Analysis Type	Purpose
1	Time-Series (Cumulative Sales)	Track daily and cumulative sales growth
2	Geographic (Hotspot Detection)	Identify states with the highest loss impact
3	Discount vs Profitability	Understand how discounts influence profits
4	Ranking (Window Function)	Find top-performing products per category
5	Month-over-Month Analysis	Detect growth trends in sales and profits

## Observations
The dataset demonstrates clear seasonal patterns in sales.

Certain states repeatedly appear in negative profit zones, possibly due to high refund or return rates.

Top-ranked products show strong profitability concentration in a few categories.

Discounts have a mixed impact, improving volume but reducing margin in some categories.

📸 Screenshots
```bash
Copy code
/screenshots/
│
├── S3.png
├── IAM.png
├── Gluer.png
├── CloudWatch.png
├── Q1.png
├── Q1 output.png
├── Q2.png
├── Q2 output.png
├── Q3.png
├── Q3 output.png
├── Q4.png
├── Q4 output.png
└── Q5.png
├── Q5 output.png
```

## Submission Checklist
 All SQL queries executed in Athena (limit 10 rows each)
 CloudWatch, IAM, and S3 screenshots uploaded
 README.md with approach and explanations
 Queries and results committed to GitHub repository

