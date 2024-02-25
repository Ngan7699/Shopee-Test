# Table Name: ***my_order_trans***
## Description: ***Table records all orders/transactions at product variation level from the beginning of time***
![image](https://github.com/Ngan7699/Shopee-Test/assets/125182638/0fd624ba-891d-46c9-9cdc-602b9a82f7cf)
# Table Name: ***my_buyer_profile***
## Description: ***Table records all buyer details***
![image](https://github.com/Ngan7699/Shopee-Test/assets/125182638/422a262e-8cf9-4074-97db-bb882a947e32)

### Q1: Find the lifetime total orders, total spent (gmv), unique items bought, earliest purchase date, last purchased date, average amount spent per order and average purchase price for the following buyer IDs and their purchased products’ main categories:
Q1: Expected outputL
- `Userid`
- `Main Category`
- `Total Orders`
- `Total Spent (RM)`
- `Unique Item Bought`
- `Earliest Purchase Date`
- `Last Purchase Date`
- `Avg Spending per Order`
- `Avg Purchase Price`

````sql
select 
    buyer_id as user_id,
    l1_cat as main_cat, 
    count(distinct order_id) as total_orders,
    sum(gmv) as total_spent,
    count(distinct product_id) as unique_items_bought,
    max(order_date) as latest_purchase_date,
    min(order_date) as earliest_purchase_date,
    round(sum(gmv)/count(distinct order_id),2) as avg_spent_order,
    round((sum(gmv)-sum(shipping_fee_paid))/sum(qty_sold),2) as avg_purchased_price
from 
    my_order_trans
where 
    buyer_id in (576123,123152)
group by 
    buyer_id, l1_cat

````
# Q2: Find out the top 10 cross border items with the highest quantity sold last month, together with their tier **, minimum selling price, total spent (gmv) and total orders.

 ***Tier is a user-defined item attribute dimension with 3 unique values:***
- Short Tail (>20 average daily orders)
- Mid Tail (between 10 - 20 average daily orders)
- Long Tail (< 10 average daily orders)
````sql
WITH tier AS (
    SELECT 
        product_id,
        CASE
            WHEN AVG(qty_sold) OVER (PARTITION BY product_id) > 20 THEN 'Short-tail'
            WHEN AVG(qty_sold) OVER (PARTITION BY product_id) BETWEEN 10 AND 20 THEN 'Mid-tail'
            ELSE 'Long-tail'
        END AS tier
    FROM order_trans
    WHERE is_cross_border = 1
)
SELECT TOP 10
    a.product_id,
    a.product_name,
    a.l1_cat AS category,
    b.tier,
    MIN(a.price) AS min_selling_price,
    SUM(a.qty_sold) AS total_qty_sold,
    SUM(a.gmv) AS total_gmv,
    COUNT(DISTINCT a.order_id) AS total_orders
FROM 
    order_trans a
JOIN 
    tier b ON a.product_id = b.product_id
WHERE 
    a.is_cross_border = 1
    AND a.order_date >= (
        SELECT DATEADD(MONTH, DATEDIFF(MONTH, 0, MAX(order_date)) - 1, 0) 
        FROM order_trans 
        WHERE is_cross_border = 1
    )
    AND a.order_date < ( 
        SELECT DATEADD(MONTH, DATEDIFF(MONTH, 0, MAX(order_date)), 0) 
        FROM order_trans 
        WHERE is_cross_border = 1
    ) 
GROUP BY 
    a.product_id, 
    a.product_name, 
    a.l1_cat,
    b.tier
ORDER BY 
    SUM(a.qty_sold) DESC;
````
# Q3: The following buyers purchased on Shopee on separate days and on several occasions:
- 123456
- 987654
- 34567

***Find the average time (in hrs) between their first and second checkout in the last 120 days.***
````sql
with check_out as 
(SELECT buyer_id,
             order_date,
             ROW_NUMBER () OVER( PARTTITION BY buyer_id ORDER BY order_date) as checktime
FROM order_trans
WHERE buyer_id in (123456,987654,34556)
AND order_date >= (SELECT DATEADD(DAY, DATEDIFF(DAY, 0, MAX(order_date)) -120,0) FROM order_trans)),
with cte1 as
(SELECT buyer_id,
            order_date as check_out1,
            LEAD(order_date,1,0) OVER (PARTTITION BY buyer_id  ORDER BY order_date) as check_out2
FROM check_out
WHERE check_time=2)
SELECT buyer_id,
            DATEDIFF(HOUR, check_out1, check_out2) as avg_time_buying
FROM  cte1
````
