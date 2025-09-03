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

---

##  Dataset Description & Data Structure (DD & DS)

### 📊 Data Description  
- **Dataset:** Simulated bicycle sales & operations data (AdventureWorks2019-style)  
- **Format:** `.sql` (Queried directly in BigQuery)
- **Time range:** 2011 - 2014

---
## ⚒️ Primary Process  
### ❶ **Business Need:**  
The sales team needs to understand product performance over the last 12 months to identify top-performing subcategories for better resource allocation.  

### **Business Question**  
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
from `adventureworks2019.Sales.SalesOrderDetail` )--2013-06-30
group by 1,2
order by 2,1;
```

### **Result:**  
| period    | Name               | qty_item | total_sales   | order_cnt |
|-----------|--------------------|---------:|--------------:|----------:|
| Sep 2013  | Bike Racks          |      312 | 22828.5120    |        71 |
| Sep 2013  | Bike Stands         |       26 | 4134.0000     |        26 |
| Sep 2013  | Bottles and Cages   |      803 | 4676.5628     |       380 |
| Sep 2013  | Bottom Brackets     |       60 | 3118.1400     |        19 |
| Sep 2013  | Brakes              |      100 | 6390.0000     |        29 |
| Sep 2013  | Caps                |      440 | 2879.4826     |       203 |
| Sep 2013  | Chains              |       62 | 752.9280      |        24 |
| Sep 2013  | Cleaners            |      296 | 1611.8300     |       108 |
| Sep 2013  | Cranksets           |       75 | 13955.8500    |        20 |
| ...       | ...                 |      ... | ...           |       ... |

### **Key Takeaway:**  

### ❷ **Business Need:**  
Management wants to evaluate year-over-year growth to identify the fastest-growing product subcategories.  

### **Business Question**  
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

### **Business Question**  
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
| yearr | Name | total_cost |
| --- | --- | --- |
| 2012 | Helmets | 149.71669 |
| 2013 | Helmets | 543.21975 |

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
