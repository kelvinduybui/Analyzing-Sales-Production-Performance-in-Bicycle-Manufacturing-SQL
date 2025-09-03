# Analyzing Sales & Production Performance in Bicycle Manufacturing using SQL  

**Author:** Bùi Xuân Bảo Duy (Kelvin)  

**Date:** March 2025  

**Tools Used:** SQL  

---

## 🗂️ Table of Contents

1️ [Context]  
2️⃣ [Dataset Description & Data Structure (DD & DS)]  
3️⃣ [Primary Process]  
4️⃣ [Insights & Recommendations]  

---
## 📘 Context

📘 **Main Context**  
- **Company Industry:** Bicycle manufacturing  
- **Business Model:** Produces and sells bicycles through multiple sales channels (retailers, online platforms, and distributors)  
- **Operations:** Manages production, inventory, and distribution across several territories  
- **Customers:** Includes both individual consumers and wholesale partners  
- **Sales Goals:** Expand market share, improve sales performance, and increase customer retention  
- **Challenges:** Fluctuating demand, inventory imbalances, and the need for data-driven decision-making in sales and supply chain  

👥 **Target Audience**  
- Sales & Marketing Teams
- Inventory & Production Planners
- Business Analysts & Data Teams

🎯 **Objective**  
This project leverages **SQL** on **Google BigQuery** to empower the sales and inventory management team of a fictional bicycle manufacturing company with data-driven insights. Through SQL queries on BigQuery, we aim to:  
- Understand performance over time and by category  
- Evaluate growth, trends, and inventory  
- Optimize sales, inventory, and marketing operations  
- Support strategic decision-making based on data

🛠️ **Tools Used**  
- **Google BigQuery (SQL):** For data storage, querying, and performing all analytical calculations.

---

##  Dataset Description & Data Structure (DD & DS)

### 📊 Data Description  
- **Dataset:** Simulated bicycle sales & operations data (AdventureWorks2019-style)  
- **Format:** `.sql` (Queried directly in BigQuery)
- **Time range:** 2011 - 2014

### 🏷️ Data Structure
**Key Tables Used:**
- `Sales.SalesOrderDetail`
- `Production.Product`
- `Production.ProductSubcategory`
- `Sales.SalesTerritory`
- `Sales.SalesOrderHeader`
- `Person.Person`
- `Person.Address`
- `Production.ProductInventory`

---
## ⚒️ Primary Process  
### ❶ **Business Need:**  
The sales team needs to understand product performance over the last 12 months to identify top-performing subcategories for better resource allocation.  

### **Business Question:**  
Calculate the total **Quantity of items**, **Sales value**, and **Order quantity** for each Subcategory in the last 12 months (L12M).  

### **SQL Query**  
```sql
select format_datetime('%b %Y', a.ModifiedDate) month
      ,c.Name
      ,sum(a.OrderQty) qty_item
      ,sum(a.LineTotal) total_sales
      ,count(distinct a.SalesOrderID) order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a 
left join `adventureworks2019.Production.Product` b
  on a.ProductID = b.ProductID
left join `adventureworks2019.Production.ProductSubcategory` c
  on b.ProductSubcategoryID = cast(c.ProductSubcategoryID as string)

where date(a.ModifiedDate) >=  (select date_sub(date(max(a.ModifiedDate)), INTERVAL 12 month)
from `adventureworks2019.Sales.SalesOrderDetail` )
group by 1,2
order by 2,1;
```

### **Result:**  
| month | Name       | qty_item | total_sales         | order_cnt |
|-------|------------|---------:|--------------------:|----------:|
| Apr 2013 | Bib-Shorts | 280      | 14833.123692        | 34        |
| Apr 2014 | Bib-Shorts | 4        | 233.974             | 1         |
| Aug 2012 | Bib-Shorts | 198      | 10659.531476        | 22        |
| Dec 2012 | Bib-Shorts | 157      | 8477.058000         | 29        |
| Feb 2013 | Bib-Shorts | 211      | 11361.453476        | 29        |
| Feb 2014 | Bib-Shorts | 4        | 233.974             | 2         |
| Jan 2013 | Bib-Shorts | 121      | 6533.274000         | 21        |
| Jul 2012 | Bib-Shorts | 436      | 23143.997159        | 44        |
| Jul 2013 | Bib-Shorts | 2        | 116.987             | 1         |
| Jun 2012 | Bib-Shorts | 490      | 26180.538728        | 60        |
| ...      | ...        | ...      | ...                 | ...       |


### **Key Takeaway:**  

### ❷ **Business Need:**  
Management wants to evaluate year-over-year growth to identify the fastest-growing product subcategories.  

### **Business Question:**  
Calculate the **YoY growth rate** for each **SubCategory** and list the **top 3** with the **highest growth** using **quantity_item**. Round to 2 decimals.

### **SQL Query**  
```sql
with 
sale_info as (
  SELECT 
      FORMAT_TIMESTAMP("%Y", a.ModifiedDate) as yr
      , c.Name
      , sum(a.OrderQty) as qty_item

  FROM `adventureworks2019.Sales.SalesOrderDetail` a 
  LEFT JOIN `adventureworks2019.Production.Product` b on a.ProductID = b.ProductID
  LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c on cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID

  GROUP BY 1,2
  ORDER BY 2 asc , 1 desc
),

sale_diff as (
  select *
  , lead (qty_item) over (partition by Name order by yr desc) as prv_qty
  , round(qty_item / (lead (qty_item) over (partition by Name order by yr desc)) -1,2) as qty_diff
  from sale_info
  order by 5 desc 
),

rk_qty_diff as (
  select *
      ,dense_rank() over( order by qty_diff desc) dk
  from sale_diff
)

select distinct Name
      , qty_item
      , prv_qty
      , qty_diff
from rk_qty_diff 
where dk <=3
order by dk ;
```

### ❸ **Business Need:**  
The company wants to identify top-performing territories for strategic sales planning.  

### **Business Question:**  
Rank the top 3 TerritoryIDs with the highest Order Quantity each year. Ties should not skip rank numbers.  

### **SQL Query**  
```sql
with raw_data as
(select extract (year from a.ModifiedDate) yr,
TerritoryID, 
sum(OrderQty) order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a
left join `adventureworks2019.Sales.SalesOrderHeader` b
using(SalesOrderID)
group by yr, TerritoryID)
,

ranked as
(select *,
dense_rank() over(
  partition by yr
  order by order_cnt desc
) rk
from raw_data)

select *
from ranked
where rk <=3
order by yr desc, rk asc;
```

### **Results:**  
| yr   | TerritoryID | order_cnt | rk |
|------|-------------|-----------|----|
| 2014 | 4           | 11632     | 1  |
| 2014 | 6           | 9711      | 2  |
| 2014 | 1           | 8823      | 3  |
| 2013 | 4           | 26682     | 1  |
| 2013 | 6           | 22553     | 2  |
| 2013 | 1           | 17452     | 3  |
| 2012 | 4           | 17553     | 1  |
| 2012 | 6           | 14412     | 2  |
| 2012 | 1           | 8537      | 3  |
| 2011 | 4           | 3238      | 1  |
| 2011 | 6           | 2705      | 2  |
| 2011 | 1           | 1964      | 3  |

### **Key Takeaway:**  

### ❹ **Business Need:**  
Marketing wants to track seasonal discount costs across subcategories to evaluate promotional impact.  

### **Business Question:**  
Calculate total seasonal discount cost for each subcategory.  

### **SQL Query**  
```sql
WITH sales_data AS (
    SELECT DISTINCT
        a.ModifiedDate,
        c.Name,
        d.DiscountPct,
        d.Type,
        a.OrderQty * d.DiscountPct * a.UnitPrice AS disc_cost
    FROM `adventureworks2019.Sales.SalesOrderDetail` a
    LEFT JOIN `adventureworks2019.Production.Product` b
        ON a.ProductID = b.ProductID
    LEFT JOIN `adventureworks2019.Production.ProductSubcategory` c
        ON CAST(b.ProductSubcategoryID AS INT64) = c.ProductSubcategoryID
    LEFT JOIN `adventureworks2019.Sales.SpecialOffer` d
        ON a.SpecialOfferID = d.SpecialOfferID
    WHERE LOWER(d.Type) LIKE '%seasonal discount%'
)
SELECT 
    EXTRACT(YEAR FROM ModifiedDate) AS year,
    Name,
    SUM(disc_cost) AS total_cost
FROM sales_data
GROUP BY 1, 2
ORDER BY 1, 2;
```

### **Results:** 
| year | Name | total_cost |
| --- | --- | --- |
| 2012 | Helmets | 149.71669 |
| 2013 | Helmets | 543.21975 |

### **Key Takeaway:**  

### ❺ **Business Need:**  
The company wants to understand customer loyalty after successful shipments in 2014.  

### **Business Question:**  
Calculate the retention rate for customers in 2014 using cohort analysis with status = 'Successfully Shipped'.  

### **SQL Query**  
```sql
with cus_info as
(select extract (month from ModifiedDate) mth,
extract (year from ModifiedDate) yr,
CustomerID
from `adventureworks2019.Sales.SalesOrderHeader`
where status = 5
and extract (year from ModifiedDate) = 2014)
,

add_row_num as
(select *,
row_number() over(
  partition by customerid
  order by mth
) row_num
from cus_info),

first_order as
(select *
from add_row_num
where row_num = 1),

add_month_diff as
(select ci.mth month_num, 
ci.yr,
CustomerID,
fo.mth month_join,
concat('M - ', ci.mth-fo.mth) month_diff
from cus_info ci
left join first_order fo
using(CustomerID))

select month_join,
month_diff,
count(distinct CustomerID) customer_cnt
from add_month_diff
group by 1,2
order by 1,2;
```

### **Result:**  
| month_join | month_diff | customer_cnt |
|------------|------------|--------------|
| 1          | M - 0       | 2076         |
| 1          | M - 1       | 78           |
| 1          | M - 2       | 89           |
| 1          | M - 3       | 252          |
| 1          | M - 4       | 96           |
| 1          | M - 5       | 61           |
| 1          | M - 6       | 18           |
| 2          | M - 0       | 1805         |
| 2          | M - 1       | 51           |
| ...        | ...         | ...          |

### **Key Takeaway:**  

### ❻ **Business Need:**  
Inventory planners need to monitor stock trends and monthly changes to prevent stockouts/overstock.  

### **Business Question:**  
Analyze the trend of stock levels and month-over-month % changes for all products in 2011. If growth rate is null, set to 0.  

### **SQL Query**  
```sql
with raw_data as
(select Name,
extract (month from EndDate) mth,
extract (year from EndDate) yr,
sum(stockedQty) stock_qty
from `adventureworks2019.Production.Product`
left join `adventureworks2019.Production.WorkOrder`
using(ProductID)
where extract (year from EndDate) = 2011
group by 1,2,3
order by 1,2 desc,3),

add_lag as
(select *, 
lag(stock_qty) over(
  partition by Name
  order by yr,mth 
) stock_prv,
from raw_data
order by name, mth desc, yr)

select *, 
coalesce(round((stock_qty-stock_prv)*100.0/stock_prv,1),0) diff
from add_lag;
```

### **Result:**  
| Name            | mth | yr   | stock_qty | stock_prv | diff   |
|-----------------|-----|------|-----------|-----------|--------|
| BB Ball Bearing | 12  | 2011 | 8475      | 14544     | -41.7  |
| BB Ball Bearing | 11  | 2011 | 14544     | 19175     | -24.2  |
| BB Ball Bearing | 10  | 2011 | 19175     | 8845      | 116.8  |
| BB Ball Bearing | 9   | 2011 | 8845      | 9666      | -8.5   |
| BB Ball Bearing | 8   | 2011 | 9666      | 12837     | -24.7  |
| BB Ball Bearing | 7   | 2011 | 12837     | 5259      | 144.1  |
| BB Ball Bearing | 6   | 2011 | 5259      | null      | 0.0    |
| Blade           | 12  | 2011 | 1842      | 3598      | -48.8  |
| Blade           | 11  | 2011 | 3598      | 4670      | -23.0  |
| ...             | ... | ...  | ...       | ...       | ...    |

### **Key Takeaway:**  

### ❼ **Business Need:**   
Management needs to understand the relationship between stock and sales to optimize inventory turnover.  

### **Business Question:**  
Calculate Stock/Sales ratio for each product by month in 2011, order by month desc and ratio desc. Round to 1 decimal.  

### **SQL Query**  
```sql
with 
sale_info as (
  select 
      extract(month from a.ModifiedDate) as mth 
     , extract(year from a.ModifiedDate) as yr 
     , a.ProductId
     , b.Name
     , sum(a.OrderQty) as sales
  from `adventureworks2019.Sales.SalesOrderDetail` a 
  left join `adventureworks2019.Production.Product` b 
    on a.ProductID = b.ProductID
  where FORMAT_TIMESTAMP("%Y", a.ModifiedDate) = '2011'
  group by 1,2,3,4
), 

stock_info as (
  select
      extract(month from ModifiedDate) as mth 
      , extract(year from ModifiedDate) as yr 
      , ProductId
      , sum(StockedQty) as stock_cnt
  from `adventureworks2019.Production.WorkOrder`
  where FORMAT_TIMESTAMP("%Y", ModifiedDate) = '2011'
  group by 1,2,3
)

select
      a.*
    , b.stock_cnt as stock  --(*)
    , round(coalesce(b.stock_cnt,0) / sales,2) as ratio
from sale_info a 
full join stock_info b 
  on a.ProductId = b.ProductId
and a.mth = b.mth 
and a.yr = b.yr
order by 1 desc, 7 desc;
```

### **Result:**  
| yr   | ProductID | Name                               | sales | stock | ratio  |
|------|-----------|------------------------------------|-------|-------|--------|
| 12   | 2011      | 745 HL Mountain Frame - Black, 48   | 1     | 27    | 27.0   |
| 12   | 2011      | 743 HL Mountain Frame - Black, 42   | 1     | 26    | 26.0   |
| 12   | 2011      | 748 HL Mountain Frame - Silver, 38  | 2     | 32    | 16.0   |
| 12   | 2011      | 722 LL Road Frame - Black, 58       | 4     | 47    | 11.75  |
| 12   | 2011      | 747 HL Mountain Frame - Black, 38   | 3     | 31    | 10.33  |
| 12   | 2011      | 726 LL Road Frame - Red, 48         | 5     | 36    | 7.2    |
| 12   | 2011      | 738 LL Road Frame - Black, 52       | 10    | 64    | 6.4    |
| 12   | 2011      | 730 LL Road Frame - Red, 62         | 7     | 38    | 5.43   |
| 12   | 2011      | 741 HL Mountain Frame - Silver, 48  | 5     | 27    | 5.4    |
| ...  | ...       | ...                                | ...   | ...   | ...    |


### **Key Takeaway:**  


### ❽ **Business Need:**  
Operations team needs to identify bottlenecks in order processing.  

### **Business Question:**  
Calculate the number and value of orders with status = 'Pending' in 2014.  

### **SQL Query**  
```sql
SELECT extract (year from OrderDate) yr,
status,
count(distinct PurchaseOrderID) order_Cnt,
sum(TotalDue) value
FROM `adventureworks2019.Purchasing.PurchaseOrderHeader`
where status =1
and extract (year from OrderDate) = 2014
group by 1,2;
```

### **Results:**
| year | Status | order | value |
| --- | --- | --- | --- |
| 2014 | 1 | 224 | 3873579.012300... |

### **Key Takeaway:**  

## 4️⃣ Insights & Recommendations  
After querying things, here are some recommendations for **Sales and Inventory** team:
- 
- 
