# [SQL] Advanture Works
## I. Project Overview
This repository contains SQL queries and analyses based on the AdventureWorks dataset available on Google BigQuery. The primary purpose of this project is to explore sales data comprehensively, providing insights into business performance, product sales trends, customer behavior, and revenue streams.
## II. Dataset Description
The dataset used is the AdventureWorks dataset, publicly available on Google BigQuery. It includes:
- Sales Data: Information about sales orders, quantities, prices, discounts, and transaction dates.
- Customer Data: Demographic and behavioral information about customers.
- Product Data: Details about product categories, pricing, and stock information.
- Employee and Territory Data: Information about sales territories and salespersons.
You can explore the dataset directly on [Google BigQuery AdventureWorks Public Dataset](https://console.cloud.google.com/bigquery?referrer=search&inv=1&invt=Ab1Sww&project=dacproject2&ws=!1m36!1m3!8m2!1s432440198297!2s3b4edf912a7c4a2c875fd19ec004dd53!1m3!8m2!1s432440198297!2s461b4568c03e445b93cf2f86b5dd9769!1m3!8m2!1s432440198297!2sf1a2c9703b504e459215ba2566a6ee8e!1m3!8m2!1s432440198297!2sd7307120612a4ef58e77a9d54ef08ea9!1m3!8m2!1s432440198297!2s00cd43487a2f4cf1b368260aaaba26a1!1m3!8m2!1s432440198297!2s88539e41e6734bdab3ba42a3e7120c23!1m3!8m2!1s432440198297!2s5425cdc4dc2b4097ba4d2d4f7c027124!1m3!8m2!1s432440198297!2s5a28656c715a48c591ddf857b7444ccf!1m3!3m2!1sadventureworks2019!2sHumanResources).
## III. Project Objectives
- Analyze total sales performance and trends over time.
- Understand customer purchasing patterns and preferences.
- Evaluate product performance, highlighting top-selling and underperforming products.
- Identify seasonal sales trends and revenue fluctuations.
- Assess performance by sales territories and salespersons.
## IV. Requirements
- Access to Google BigQuery
- Basic to intermediate SQL knowledge
## V. Explore Data set
### Query 01: Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M
- SQL query:
<pre>
select
  format_timestamp('%b %Y', a.ModifiedDate) as period,
  c.name as name,
  sum(a.OrderQty) AS qty_item,
  sum(a.LineTotal) AS total_sales,
  count(distinct a.SalesOrderID) as order_cnt
from `adventureworks2019.Sales.SalesOrderDetail` as a
left join `adventureworks2019.Production.Product` as b
  on a.ProductID = b.ProductID
left join `adventureworks2019.Production.ProductSubcategory` AS c
  on cast(b.ProductSubcategoryID as int64) = c.ProductSubcategoryID
where date(a.ModifiedDate) >= date_sub(
  date_trunc((select date(max(ModifiedDate)) from `adventureworks2019.Sales.SalesOrderDetail`), MONTH),
  interval 11 MONTH
)
group by 1, 2
order by 1 desc, 2;
</pre>
- Result:
<pre>
| period   | name              | qty_item | total_sales | order_cnt |
|----------|-------------------|----------|-------------|-----------|
| Sep 2013 | Bike Racks        | 312      | 22828.51    | 71        |
| Sep 2013 | Bike Stands       | 26       | 4134.00     | 26        |
| Sep 2013 | Bottles and Cages | 803      | 4676.56     | 380       |
| Sep 2013 | Bottom Brackets   | 60       | 3118.14     | 19        |
| Sep 2013 | Brakes            | 100      | 6390.00     | 29        |
</pre>
### Query 02: Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Round results to 2 decimal
- SQL query:
<pre>
with 
sale_info as (
  select 
      format_timestamp("%Y", a.ModifiedDate) as yr
      , c.Name
      , sum(a.OrderQty) as qty_item

  from `adventureworks2019.Sales.SalesOrderDetail` a 
  left join `adventureworks2019.Production.Product` b on a.ProductID = b.ProductID
  left join `adventureworks2019.Production.ProductSubcategory` c on cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID
  group by 1,2
  order by 2 asc , 1 desc
),

sale_diff as (
  select 
  yr
  ,Name
  ,qty_item
  ,lead (qty_item) over (partition by Name order by yr desc) as prv_qty
  ,round(qty_item / (lead (qty_item) over (partition by Name order by yr desc)) -1,2) as qty_diff
  from sale_info
  order by 5 desc 
),

rk_qty_diff as (
  select 
    yr
    ,Name
    ,qty_item
    ,prv_qty
    ,qty_diff
    ,dense_rank() over( order by qty_diff desc) dk
  from sale_diff
)

select distinct Name
      , qty_item
      , prv_qty
      , qty_diff
      ,dk
from rk_qty_diff 
where dk <=3
order by dk ;
</pre>
- Result:
<pre>
| Name             | qty_item | prv_qty | qty_diff | dk |
|------------------|----------|---------|----------|----|
| Mountain Frames  | 3168     | 510     | 5.21     | 1  |
| Socks            | 2724     | 523     | 4.21     | 2  |
| Road Frames      | 5564     | 1137    | 3.89     | 3  |
</pre>
### Query 03: Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number
- SQL query:
<pre>
with 
sale_info as (
  select 
      FORMAT_TIMESTAMP("%Y", a.ModifiedDate) as yr
      , b.TerritoryID
      , sum(OrderQty) as order_cnt 
  from `adventureworks2019.Sales.SalesOrderDetail` a 
  LEFT JOIN `adventureworks2019.Sales.SalesOrderHeader` b 
    on a.SalesOrderID = b.SalesOrderID
  group by 1,2
),

sale_rank as (
  select 
      yr
      ,TerritoryID
      ,order_cnt
      ,dense_rank() over (partition by yr order by order_cnt desc) as rk 
  from sale_info 
)

select yr
    , TerritoryID
    , order_cnt
    , rk
from sale_rank 
where rk in (1,2,3);   --rk <=3
</pre>
- Result:
<pre>
| yr   | TerritoryID | order_cnt | rk |
|------|-------------|-----------|----|
| 2011 | 4           | 3238      | 1  |
| 2011 | 6           | 2705      | 2  |
| 2011 | 1           | 1964      | 3  |
| 2014 | 4           | 11632     | 1  |
| 2014 | 6           | 9711      | 2  |
| 2014 | 1           | 8823      | 3  |
| 2012 | 4           | 17553     | 1  |
| 2012 | 6           | 14412     | 2  |
| 2012 | 1           | 8537      | 3  |
| 2013 | 4           | 26682     | 1  |
| 2013 | 6           | 22553     | 2  |
| 2013 | 1           | 17452     | 3  |
</pre>
### Query 04: Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory
- SQL query
<pre>
select 
    format_timestamp("%Y", ModifiedDate)
    , Name
    , sum(disc_cost) as total_cost
from (
      select distinct a.ModifiedDate
      , c.Name
      , d.DiscountPct, d.Type
      , a.OrderQty * d.DiscountPct * UnitPrice as disc_cost 
      from `adventureworks2019.Sales.SalesOrderDetail` a
      left join `adventureworks2019.Production.Product` b on a.ProductID = b.ProductID
      left join `adventureworks2019.Production.ProductSubcategory` c on cast(b.ProductSubcategoryID as int) = c.ProductSubcategoryID
      left join `adventureworks2019.Sales.SpecialOffer` d on a.SpecialOfferID = d.SpecialOfferID
      where lower(d.Type) like '%seasonal discount%' 
)
group by 1,2;
</pre>
- Result:
<pre>
| f0_  | Name    | total_cost |
|------|---------|------------|
| 2012 | Helmets | 149.71669  |
| 2013 | Helmets | 543.21975  |
</pre>
### Query 05: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)
- SQL query
<pre>
with 
info as (
  select  
      extract(month from ModifiedDate) as month_no
      , extract(year from ModifiedDate) as year_no
      , CustomerID
      , count(Distinct SalesOrderID) as order_cnt
  from `adventureworks2019.Sales.SalesOrderHeader`
  where FORMAT_TIMESTAMP("%Y", ModifiedDate) = '2014'
  and Status = 5
  group by 1,2,3
  order by 3,1 
),

row_num as (--đánh số thứ tự các tháng họ mua hàng
  select *
      , row_number() over (partition by CustomerID order by month_no) as row_numb
  from info 
), 

first_order as (   --lấy ra tháng đầu tiên của từng khách
  select *
  from row_num
  where row_numb = 1
), 

month_gap as (
  select 
      a.CustomerID
      , b.month_no as month_join
      , a.month_no as month_order
      , a.order_cnt
      , concat('M - ',a.month_no - b.month_no) as month_diff
  from info a 
  left join first_order b 
  on a.CustomerID = b.CustomerID
  order by 1,3
)

select month_join
      , month_diff 
      , count(distinct CustomerID) as customer_cnt
from month_gap
group by 1,2
order by 1,2;
</pre>
- Result:
<pre>
| month_join | month_diff | customer_cnt |
|------------|------------|---------------|
| 1          | M - 0      | 2076          |
| 1          | M - 1      | 78            |
| 1          | M - 2      | 89            |
| 1          | M - 3      | 252           |
| 1          | M - 4      | 96            |
| 1          | M - 5      | 61            |
| 1          | M - 6      | 18            |
| 2          | M - 0      | 1805          |
| 2          | M - 1      | 51            |
| 2          | M - 2      | 61            |
| 2          | M - 3      | 234           |
| 2          | M - 4      | 58            |
| 2          | M - 5      | 8             |
| 3          | M - 0      | 1918          |
| 3          | M - 1      | 43            |
| 3          | M - 2      | 58            |
| 3          | M - 3      | 44            |
| 3          | M - 4      | 11            |
| 4          | M - 0      | 1906          |
| 4          | M - 1      | 34            |
| 4          | M - 2      | 44            |
| 4          | M - 3      | 7             |
| 5          | M - 0      | 1947          |
| 5          | M - 1      | 40            |
| 5          | M - 2      | 7             |
| 6          | M - 0      | 909           |
| 6          | M - 1      | 10            |
| 7          | M - 0      | 148           |
</pre>
### Query 06: Trend of Stock level & MoM diff % by all product in 2011. Round to 1 decimal
- SQL query:
<pre>
with 
raw_data as (
  select
      extract(month from a.ModifiedDate) as mth 
      , extract(year from a.ModifiedDate) as yr 
      , b.Name
      , sum(StockedQty) as stock_qty
  from `adventureworks2019.Production.WorkOrder` a
  left join `adventureworks2019.Production.Product` b on a.ProductID = b.ProductID
  where FORMAT_TIMESTAMP("%Y", a.ModifiedDate) = '2011'
  group by 1,2,3
  order by 1 desc 
)

select  Name
      , mth, yr 
      , stock_qty
      , stock_prv
      , round(coalesce((stock_qty /stock_prv -1)*100 ,0) ,1) as diff
from (
      select *
      , lead (stock_qty) over (partition by Name order by mth desc) as stock_prv
      from raw_data
      )
order by 1 asc, 2 desc;
</pre>
- Result (top 10)
<pre>
| Name            | mth | yr   | stock_qty | stock_prv | diff   |
|-----------------|-----|------|-----------|-----------|--------|
| BB Ball Bearing | 12  | 2011 | 8475      | 14544     | -41.7  |
| BB Ball Bearing | 11  | 2011 | 14544     | 19175     | -24.2  |
| BB Ball Bearing | 10  | 2011 | 19175     | 8845      | 116.8  |
| BB Ball Bearing | 9   | 2011 | 8845      | 9666      | -8.5   |
| BB Ball Bearing | 8   | 2011 | 9666      | 12837     | -24.7  |
| BB Ball Bearing | 7   | 2011 | 12837     | 5259      | 144.1  |
| BB Ball Bearing | 6   | 2011 | 5259      |           | 0      |
| Blade           | 12  | 2011 | 1842      | 3598      | -48.8  |
| Blade           | 11  | 2011 | 3598      | 4670      | -23    |
</pre>
### Query 07: Calc Ratio of Stock / Sales in 2011 by product name, by month. Order results by month desc, ratio desc. Round Ratio to 1 decimal mom yoy
- SQL query:
<pre>
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
</pre>
- Result (top 10):
<pre>
| mth | yr   | ProductId | Name                                 | sales  | stock  | ratio  |
|-----|------|-----------|--------------------------------------|--------|--------|--------|
| 12  | 2011 | 745       | HL Mountain Frame - Black, 48        | 1      | 27     | 27.00  |
| 12  | 2011 | 743       | HL Mountain Frame - Black, 42        | 1      | 26     | 26.00  |
| 12  | 2011 | 748       | HL Mountain Frame - Silver, 38       | 2      | 32     | 16.00  |
| 12  | 2011 | 722       | LL Road Frame - Black, 58            | 4      | 47     | 11.75  |
| 12  | 2011 | 747       | HL Mountain Frame - Black, 38        | 3      | 31     | 10.33  |
| 12  | 2011 | 726       | LL Road Frame - Red, 48              | 5      | 36     | 7.20   |
| 12  | 2011 | 738       | LL Road Frame - Black, 52            | 10     | 64     | 6.40   |
| 12  | 2011 | 730       | LL Road Frame - Red, 62              | 7      | 38     | 5.43   |
| 12  | 2011 | 741       | HL Mountain Frame - Silver, 48       | 5      | 27     | 5.40   |
| 12  | 2011 | 725       | LL Road Frame - Red, 44              | 12     | 53     | 4.42   |
</pre>
### Query 08: No of order and value at Pending status in 2014
- SQL query:
<pre>
select 
    extract (year from ModifiedDate) as yr
    , Status
    , count(distinct PurchaseOrderID) as order_Cnt 
    , sum(TotalDue) as value
from `adventureworks2019.Purchasing.PurchaseOrderHeader`
where Status = 1
and extract(year from ModifiedDate) = 2014
group by 1,2;
</pre>
- Result:
<pre>
| yr   | Status | order_Cnt | value            |
|------|--------|-----------|------------------|
| 2014 | 1      | 224       | 3873579.0123     |
</pre>
## VI. Contributing
  Contributions are welcome! Please open an issue first to discuss potential improvements or submit a pull request.
## VII. Contact
For any questions or suggestions, please contact:
- Ton The Son
- Email: tontheson@gmail.com
- LinkedIn: https://www.linkedin.com/in/tontheson/
