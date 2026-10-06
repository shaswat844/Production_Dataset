# Demand vs Availability Analysis Dashboard

### Dashboard Link: https://app.powerbi.com/links/Io3JFquUkZ?ctid=500356b3-ac75-4410-bd8f-decd38beb5a2&pbi_source=linkShare

**Tech stack:** Microsoft SQL Server | SSMS | Power BI Desktop | DAX

---

## Problem Statement

This dashboard helps business and operations teams understand how well product **availability** meets customer **demand**. It shows how much is demanded and supplied per day, how large the overall supply shortage is, and what that shortage means financially in terms of profit and loss.

**Key questions the dashboard answers:**
- On an average day, how much demand is there, and how much of it can be supplied?
- How big is the total supply shortage (demand minus availability)?
- How much profit is earned, how much is lost, and what is the average daily loss?

---

## Dataset

- **Source:** Course-provided dataset (SQL Server + Power BI project)
- **Format:** CSV imported into Microsoft SQL Server
- **Raw table:** `Prod+Env+Inventory+Dataset` (inventory / demand / availability data)
- **Lookup table:** `Products+(1)` (product name and unit price)
- **Rows after cleaning:** 1,043
- **Final reporting table:** `new_table` (loaded into Power BI as `Demand/Availability_Data`)

| Column | Description |
|---|---|
| `Order_Date_DD_MM_YYYY` | Date of the record |
| `product_id` | Unique product identifier |
| `product_name` | Product name (from the Products table) |
| `unit_price` | Price per unit (from the Products table) |
| `demand` | Quantity demanded |
| `availability` | Quantity available to supply |
| `Loss/Profit` | Calculated column in Power BI: `availability - demand` (positive = surplus / profit, negative = shortage / loss) |

---

## Architecture / Data Flow

```
Raw CSV data  -->  SQL Server (test environment)  -->  Left join + cleaning (SQL)  -->  Power BI Desktop  -->  Report (2 pages)
```

---

## Project Workflow

1. **Understood the test environment data and requirements** and defined the KPIs needed.
2. **Imported the data into SQL Server** (test environment).
3. **Applied a LEFT JOIN** between the inventory table and the products table to prepare the data for reporting.
4. **Imported the data into Power BI Desktop** from the test environment.
5. **Created the DAX measures and KPIs** for Page 1 (demand / availability / shortage).
6. **Created the DAX measures and KPIs** for Page 2 (profit / loss).
7. **Imported the data into the production environment** in SQL Server.
8. **Cleaned the data using SQL** (fixed product IDs, removed junk rows) and moved the report from test to production.

---

## SQL Queries (Microsoft SQL Server)

### Step 1: Create the database and explore the data
```sql
CREATE DATABASE PROD;

USE PROD;

SELECT * FROM [dbo].[Prod+Env+Inventory+Dataset];

-- Check for missing / blank order dates
SELECT DISTINCT Order_Date_DD_MM_YYYY
FROM [dbo].[Prod+Env+Inventory+Dataset]
WHERE Order_Date_DD_MM_YYYY IS NULL OR Order_Date_DD_MM_YYYY = '';

-- Check the product IDs
SELECT DISTINCT Product_ID
FROM [dbo].[Prod+Env+Inventory+Dataset]
ORDER BY Product_ID;
```

### Step 2: Fix incorrect Product IDs
Product IDs 21 and 22 had no match in the Products table, so they were mapped to the correct IDs (21 to 7, 22 to 11).
```sql
UPDATE [dbo].[Prod+Env+Inventory+Dataset]
SET Product_ID = 7
WHERE Product_ID = 21;

UPDATE [dbo].[Prod+Env+Inventory+Dataset]
SET Product_ID = 11
WHERE Product_ID = 22;

SELECT DISTINCT Demand FROM [dbo].[Prod+Env+Inventory+Dataset];
```

### Step 3: Join the inventory data with product details (LEFT JOIN)
```sql
SELECT * INTO new_table
FROM (
    SELECT a.[Order_Date_DD_MM_YYYY],
           a.product_id,
           a.availability,
           a.demand,
           b.product_name,
           b.unit_price
    FROM [dbo].[Prod+Env+Inventory+Dataset] AS a
    LEFT JOIN [dbo].[Products+(1)] AS b
        ON a.product_id = b.product_id
) x;
```

### Step 4: Check for junk rows (read-only)
The production load contained extra rows where both availability and demand were 0. These are not real data, so they were checked before deleting.
```sql
USE PROD;

-- 1. Total rows in the table (expected 1043 when clean)
SELECT COUNT(*) AS TotalRows
FROM [dbo].[Prod+Env+Inventory+Dataset];

-- 2. Number of junk rows
SELECT COUNT(*) AS JunkRowCount
FROM [dbo].[Prod+Env+Inventory+Dataset]
WHERE Availability = 0 AND Demand = 0;

-- 3. Preview 20 junk rows
SELECT TOP 20 *
FROM [dbo].[Prod+Env+Inventory+Dataset]
WHERE Availability = 0 AND Demand = 0;

-- 4. Preview 20 normal rows for comparison
SELECT TOP 20 *
FROM [dbo].[Prod+Env+Inventory+Dataset]
WHERE Availability > 0 OR Demand > 0;
```

### Step 5: Back up the table and delete the junk rows
```sql
USE PROD;

-- 1. Backup the table exactly as it is now
SELECT * INTO Prod_Env_Inventory_Dataset_Backup
FROM [dbo].[Prod+Env+Inventory+Dataset];

-- 2. Delete junk rows (both Availability and Demand are 0)
DELETE FROM [dbo].[Prod+Env+Inventory+Dataset]
WHERE Availability = 0 AND Demand = 0;

-- 3. Confirm the row count (should be 1043)
SELECT COUNT(*) AS RowCountAfterCleanup
FROM [dbo].[Prod+Env+Inventory+Dataset];
```

### Step 6: Validate the join (read-only)
These checks were run to find out why Total Profit looked far too high.
```sql
USE PROD;

-- 1. Product_ID data type must match in both tables
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE COLUMN_NAME = 'Product_ID'
  AND TABLE_NAME IN ('Prod+Env+Inventory+Dataset', 'Products+(1)');

-- 2. Duplicate Product IDs in the Products table (duplicates cause fan-out errors)
SELECT [Product_ID], COUNT(*) AS Occurrences
FROM [dbo].[Products+(1)]
GROUP BY [Product_ID]
HAVING COUNT(*) > 1;

-- 3. Inventory rows with no matching product (expected 88 before the ID fix: IDs 21 and 22)
SELECT COUNT(*) AS UnmatchedRows
FROM [dbo].[Prod+Env+Inventory+Dataset] i
LEFT JOIN [dbo].[Products+(1)] p
    ON i.[Product_ID] = p.[Product_ID]
WHERE p.[Product_ID] IS NULL;
```

> **Restore option:** if something goes wrong after the delete, the table can be restored from `Prod_Env_Inventory_Dataset_Backup`.

---

## DAX Measures

### Calculated Column

```DAX
Loss/Profit = 'Demand/Availability_Data'[availability] - 'Demand/Availability_Data'[demand]
```

### Measures

All measures are stored in a dedicated **Measure Table**.

```DAX
Total Demand = SUM('Demand/Availability_Data'[demand])
```

```DAX
Total_Availability = SUM('Demand/Availability_Data'[availability])
```

```DAX
Total_Num_OF_Days = DISTINCTCOUNT('Demand/Availability_Data'[Order_Date_DD_MM_YYYY].[Date])
```

```DAX
Avg_Demand_Per_Day = DIVIDE([Total Demand], [Total_Num_OF_Days])
```

```DAX
Avg_Availability_Per_Day = DIVIDE([Total_Availability], [Total_Num_OF_Days])
```

```DAX
Total_Supply_Shortage = [Total Demand] - [Total_Availability]
```

```DAX
Total_Profit =
SUMX (
    FILTER ( 'Demand/Availability_Data', 'Demand/Availability_Data'[Loss/Profit] > 0 ),
    'Demand/Availability_Data'[Loss/Profit] * 'Demand/Availability_Data'[unit_price]
)
```

```DAX
Total_Loss =
SUMX (
    FILTER ( 'Demand/Availability_Data', 'Demand/Availability_Data'[Loss/Profit] < 0 ),
    'Demand/Availability_Data'[Loss/Profit] * 'Demand/Availability_Data'[unit_price]
) * -1
```

```DAX
Average_Loss_Per_Day = DIVIDE([Total_Loss], [Total_Num_OF_Days])
```

> `Total_Loss` is multiplied by -1 so the loss is shown as a positive number.

---

## Dashboard Snapshots

### Page 1: Demand and Availability
<img width="1295" height="725" alt="Page 1 - Demand and Availability" src="https://github.com/user-attachments/assets/a70b0019-64ae-401e-8578-56efb78e856b" />

### Page 2: Profit and Loss
<img width="1297" height="725" alt="Page 2 - Profit and Loss" src="https://github.com/user-attachments/assets/51c7978a-31ad-45c0-b7de-da3e6bc619a9" />


---

## Insights

The report has two pages, each with three KPI cards.

### Page 1: Demand, Availability and Supply Shortage

| KPI | Value |
|---|---|
| Average Demand per Day | 48.65 |
| Average Availability per Day | 24.70 |
| Total Supply Shortage | 61K |

**Takeaway:** On an average day, availability (24.70) covers only about half of demand (48.65), which results in a total supply shortage of about 61K units.

### Page 2: Profit and Loss

| KPI | Value |
|---|---|
| Total Profit | 301K |
| Total Loss | 8M |
| Average Daily Loss | 2.97K |

**Takeaway:** Total loss (8M) is far larger than total profit (301K), and the business loses about 2.97K on an average day.

### Overall Takeaways
- Supply is not keeping up with demand, so a large share of demand goes unmet.
- The financial impact of that gap is much larger than the profit earned.
- Closing even part of the supply gap would have a large effect on the loss figure.

---

## Recommendations

1. Increase availability, starting with the products that have the largest gap between demand and availability.
2. Investigate which products and dates contribute most to the 8M loss.
3. Add slicers (Date, Product) and a product-wise breakdown of shortage, profit and loss for deeper analysis.
4. Monitor Average Daily Loss as a regular KPI to track improvement.

---

## Tools Used

- Microsoft SQL Server and SQL Server Management Studio (SSMS)
- Power BI Desktop
- DAX

## Author

**Shaswat Kumar** | [LinkedIn](https://www.linkedin.com/in/shaswat-kumar07) | [shaswat844@gmail.com](mail:shaswat844@gmail.com)
