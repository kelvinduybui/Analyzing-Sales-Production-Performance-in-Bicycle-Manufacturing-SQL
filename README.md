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
## 📘 Main Context
- **Company Industry:** Bicycle manufacturing  
- **Business Model:** Produces and sells bicycles through multiple sales channels (retailers, online platforms, and distributors)  
- **Operations:** Manages production, inventory, and distribution across several territories  
- **Customers:** Includes both individual consumers and wholesale partners  
- **Sales Goals:** Expand market share, improve sales performance, and increase customer retention  
- **Challenges:** Fluctuating demand, inventory imbalances, and the need for data-driven decision-making in sales and supply chain  

## 👥 Target Audience
- Sales & Marketing Teams
- Inventory & Production Planners
- Business Analysts & Data Teams

## 🎯 Objective
This project leverages **SQL** on **Google BigQuery** to empower the sales and inventory management team of a fictional bicycle manufacturing company with data-driven insights. Through SQL queries on BigQuery, we aim to:  
- Understand performance over time and by category  
- Evaluate growth, trends, and inventory  
- Optimize sales, inventory, and marketing operations  
- Support strategic decision-making based on data

---

## 2️⃣ Dataset Description & Data Structure (DD & DS)

### 📊 Data Description  
- **Dataset:** Simulated bicycle sales & operations data (AdventureWorks2019-style)  
- **Format:** `.sql` (Queried directly in BigQuery)
- **Time range:** 2011 - 2014

---
## ⚒️ Primary Process  
### 1️ **Business Need**  
> The sales team needs to understand product performance over the last 12 months to identify top-performing subcategories for better resource allocation.  

**Business Question**  
> Calculate the total **Quantity of items**, **Sales value**, and **Order quantity** for each Subcategory in the last 12 months (L12M).  

**SQL Query**  
```sql
with lastest_month as
(select *, (FORMAT_DATE('%B %Y', ModifiedDate)) AS real_year_month
, extract(year from ModifiedDate) real_year
, extract(month from ModifiedDate) real_month
from `adventureworks2019.Sales.SalesOrderDetail`
order by extract(year from ModifiedDate) desc
,extract(month from ModifiedDate) desc)
,

rank_month as
(select *,
dense_rank() over (
order by real_year desc, real_month desc
) dk
from lastest_month)

select real_year_month month
, c.Name
, sum(OrderQty) qty_item
, sum(LineTotal) total_sales
, count(distinct SalesOrderID) order_cnt
FROM rank_month a
left join `adventureworks2019.Production.Product` b
using(ProductID)
left join `adventureworks2019.Production.ProductSubcategory` c
on cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID
where dk <=12
group by month, c.Name
order by month desc, c.Name
```

**Result:**  
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

**Key Takeaway:**  
> 
