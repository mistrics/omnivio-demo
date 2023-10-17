# Here is a detailed report of possible metrics that we can use for reporting in an ecommerce sales data.

Dataset columns:
- order_id
- user_id
- product_name
- category
- order_date
- ship_date
- unit_price
- quantity
- discount
- review_score


## Question: Find most and least popular categories
### -- Step 1: Finding counts of all category items sold
```
select category, sum(quantity) as item_count
From sales_data 
GROUP BY category
ORDER BY sum(quantity) DESC;
```
| Category | Sold Items |
|-----|-----|
|Beauty |413|
|Electronics|	397|
|Toys|	382|
|Automotive|	377|
|Sports|	372|
|Books|	368|
|Home|	367|
|Clothing|	336|

### -- Step 2: Lets just select 1 row by ASC and DESC and UNION those two rows for single result.
-- There are two approaches 
    -- 1: Where we can just select items based on LIMIT 1 with ASC and DESC with a union like below
    -- BUT, this will not work effectivaly if there are two or more categories with same high and low sold item counts.
```
    (
        select category, sum(quantity) from sales_data
        group by category
        order by sum(quantity) DESC LIMIT 1
    )
    UNION
    (
        select category, sum(quantity) from sales_data
        group by category
        order by sum(quantity) ASC LIMIT 1
    );
```

| Category | Sold Items |
|-----|-----|
|Beauty |413|
|Clothing|	336|

### -- 2. There is a better approach to this where if two or more categories have same counts then results should also show them.
```
    WITH sales_data_details AS (
        select category, sum(quantity) as item_count
        from sales_data
        group by category
        ORDER BY item_count DESC
    )

    select category, item_count from sales_data_details 
    GROUP BY category, item_count
    having item_count  = (select min(item_count) from sales_data_details) or item_count  = (select max(item_count) from sales_data_details)
```

| category | item_count |
|-----|-----|
|Beauty |413|
|Clothing|	336|


# Question: We found the top and bottom one, but what if we want to find top two/n and bottom two/n categories.
```
-- here if we + (n) with min and + (n) with max then we can find more top and bottom categories
WITH sales_data_details AS (
    select category, sum(quantity) as item_count, ROW_NUMBER() OVER (ORDER BY sum(quantity) DESC) as rn
    from sales_data
    group by category
    ORDER BY item_count DESC
)
select category, item_count
from sales_data_details
group by category, item_count, rn
having rn  <= (select min(rn)+1 from sales_data_details) or rn >= (select max(rn)-1 from sales_data_details)
ORDER BY item_count DESC;
```

| category | item_count |
|-----|-----|
|Beauty |413|
|Electronics|	397|
|Home|	367|
|Clothing|	336|


# Question: Find top two products from each category

```
select * from (
select category, product_name, sum(quantity), RANK() OVER(PARTITION BY category ORDER BY sum(quantity) DESC) as rank
from sales_data
GROUP BY category, product_name
ORDER BY category, sum(quantity) DESC ) product_ranking
where rank <= 2
-- Here we can change number of rank and get more/less data.
```

|category	|product_name	|sum	|rank|
|----|----|----|----|
|Automotive	|Car Freshener	|117	|1|
|Automotive	|Wax	|80	|2|
|Beauty	|Foundation	|117	|1|
|Beauty	|Nail polish	|84	|2|
|Books	|Beloved	|99	|1|
|Books	|Ulysses	|89	|2|
|Clothing	|Dress	|89	|1|
|Clothing	|Jacket	|79	|2|
|Electronics	|Tablet	|98	|1|
|Electronics	|Headphones	|92	|2|
|Home	|Table	|81	|1|
|Home	|Chair	|76	|2|
|Home	|Sofa	|76	|2|
|Sports	|Swimsuit	|84	|1|
|Sports	|Running Shoes	|79	|2|
|Sports	|Tennis Racket	|79	|2|
|Toys	|Lego	|97	|1|
|Toys	|Board Game	|92	|2|


# Question: Enhance the data set with calculated fields for discounts, also prices with and without discount.

Our sales data is very raw and we have to enhance it with some precalculated fields to consider discounts so that we can find correct revenues, based on category, dates etc.

```
-- Here is the function that will be used to better calculations considering discounted prices.

CREATE OR REPLACE FUNCTION public.enhanced_sales_data()
RETURNS TABLE (
    order_id integer,
    product_name TEXT,
    category TEXT,
    order_date TIMESTAMP,
    ship_date TIMESTAMP,
    review_score integer,
    unit_price FLOAT,
    quantity BIGINT,
    order_value_before_discount FLOAT,
    order_value_after_discount FLOAT,
    value_per_unit_after_discount FLOAT,
    discount_per_item FLOAT,
    total_discount_value FLOAT
)
AS 
$body$
    select 
    order_id,
    product_name, category, 
    order_date::TIMESTAMP, 
    ship_date::TIMESTAMP, 
    review_score, 
    unit_price,
    quantity,
    ROUND((unit_price*quantity)::Decimal, 2) as order_value_before_discount,
    ROUND((unit_price*quantity - (unit_price*discount*.01*quantity))::Decimal, 2) as order_value_after_discount,
    ROUND((unit_price - (unit_price*discount*.01))::Decimal, 2) as value_per_unit_after_discount,
    ROUND((unit_price*discount*.01)::Decimal, 2) as discount_per_item, 
    ROUND((unit_price*discount*.01*quantity)::Decimal, 2) as total_discount_value
    from sales_data
$body$
LANGUAGE SQL;
```



# Lets find some more metric data

## Question: What is the average order value before and after discount.

```
SELECT ROUND(AVG(order_value)::Decimal, 2) as avg_order_value, ROUND(AVG(order_value_after_discount)::Decimal, 2) as avg_order_value_after_discount
FROM (
    select 
    order_id, 
    ROUND(sum(order_value_before_discount)::Decimal, 2) as order_value, 
    ROUND(sum(order_value_after_discount)::Decimal, 2) as order_value_after_discount
    from enhanced_sales_data()
    GROUP BY order_id
) order_values
```
|avg_order_value|avg_order_value_after_discount|
|----|----|
|1485.67|1332.90|

## Question: What is the average oder value (before and after discount) of each category?

```
select 
category, 
ROUND(AVG(order_value_before_discount)::Decimal, 2) as avg_order_value_before_discount,
ROUND(AVG(order_value_after_discount)::Decimal, 2) as avg_order_value_after_discount
from enhanced_sales_data()
GROUP BY category
ORDER BY category ASC
```
|category|avg_order_value_BD|avg_order_value_AD|
|--|--|--|
|Automotive|1634.68|1471.10|
|Beauty|1463.16|1306.40|
|Books|1538.31|1375.59|
|Clothing|1436.06|1278.28|
|Electronics|1413.05|1282.09|
|Home|1328.38|1187.35|
|Sports|1523.40|1364.92|
|Toys|1552.52|1399.96|


# Question: Lets do couple of use cases based on the dates.
```
-- Show daily revenue before and after discount.

select 
order_date::DATE as date_of_order, 
ROUND(sum(order_value_after_discount)::Decimal, 2) Revenue_AD, 
ROUND(sum(order_value_before_discount)::Decimal, 2) Revenue_BD
from enhanced_sales_data()
where 1=1
GROUP BY date_of_order
ORDER BY date_of_order;
```

|date_of_order|revenue_ad|revenue_bd|
|--|--|--|
|2023-01-01|43714.48|49239.54|
|2023-01-02|48335.22|53462.35|
|2023-01-03|35051.52|39075.11|
|2023-01-04|51609.49|56757.58|
|2023-01-05|55474.26|61435.84|
|2023-01-06|53838.64|60233.51|
|2023-01-07|40628.10|44715.90|
|2023-01-08|30571.06|34227.97|
|2023-01-09|44306.46|48342.84|
|2023-01-10|42762.60|47795.90|
|2023-01-11|26246.86|29462.63|
|2023-01-12|40599.67|44691.70|
|2023-01-13|44756.76|49792.81|
|2023-01-14|43928.65|47656.87|
|2023-01-15|37880.83|42385.64|
|2023-01-16|41436.18|45166.88|
|2023-01-17|31184.50|36341.41|
|2023-01-18|56268.69|63934.46|
|2023-01-19|29640.93|32504.40|
|2023-01-20|63210.13|71321.96|
|2023-01-21|44465.34|49072.08|
|2023-01-22|34015.80|38370.99|
|2023-01-23|39830.63|44295.06|
|2023-01-24|49161.88|54534.88|
|2023-01-25|34773.70|39287.51|
|2023-01-26|46603.31|51772.54|
|2023-01-27|37526.41|42219.11|
|2023-01-28|61425.96|69442.20|
|2023-01-29|62660.09|69681.08|
|2023-01-30|38603.93|43712.82|
|2023-01-31|22388.43|24738.47|

