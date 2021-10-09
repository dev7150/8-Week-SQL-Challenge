# 🐟 Case Study #6 - Clique Bait

## 👩🏻‍💻 Solution - B. Product Funnel Analysis

Using a single SQL query - create a new output table which has the following details:

**[Table 1]**
1. How many times was each product viewed?
2. How many times was each product added to cart?
3. How many times was each product added to a cart but not purchased (abandoned)?
4. How many times was each product purchased?

**[Table 2]**

Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

Use your 2 new output tables - answer the following questions:

1. Which product had the most views, cart adds and purchases?
2. Which product was most likely to be abandoned?
3. Which product had the highest view to purchase percentage?
4. What is the average conversion rate from view to cart add?
5. What is the average conversion rate from cart add to purchase?

***

## Planning Our Answer Strategy

Let us visualize the output table.

| Column | Description | 
| ------- | ----------- |
| product | Name of the product |
| views | Number of views for each product |
| cart_adds | Number of cart adds for each product |
| abandoned | Number of times product was added to a cart, but not purchased |
| purchased | Number of times product was purchased |

These information would come from several tables
- `events` table - visit_id, page_id, event_type
- `page_hierarchy` table - page_id, product_category

**Solution for [Table 1]**

- Note 1 - In `product_page_events` CTE, find page views and cart adds for individual visit ids by wrapping `SUM` around `CASE statements` so that we do not have to group the results by `event_type` as well.
- Note 2 - In `purchase_events` CTE, get only visit ids that have made purchases.
- Note 3 - In `combined_table` CTE, merge `product_page_events` and `purchase_events` using `LEFT JOIN`. Take note of the table sequence. In order to filter for visit ids with purchases, we use a `CASE statement` and where visit id is not null, it means the visit id is a purchase. 

```sql
WITH product_page_events AS ( -- Note 1
  SELECT 
    e.visit_id,
    ph.product_id,
    ph.page_name AS product_name,
    ph.product_category,
    SUM(CASE WHEN e.event_type = 1 THEN 1 ELSE 0 END) AS page_view, -- 1 for Page View
    SUM(CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) AS cart_add -- 2 for Add Cart
  FROM clique_bait.events AS e
  JOIN clique_bait.page_hierarchy AS ph
    ON e.page_id = ph.page_id
  WHERE product_id IS NOT NULL
  GROUP BY e.visit_id, ph.product_id, ph.page_name, ph.product_category
),
purchase_events AS ( -- Note 2
  SELECT 
    DISTINCT visit_id
  FROM clique_bait.events
  WHERE event_type = 3 -- 3 for Purchase
),
combined_table AS ( -- Note 3
  SELECT 
    ppe.visit_id, 
    ppe.product_id, 
    ppe.product_name, 
    ppe.product_category, 
    ppe.page_view, 
    ppe.cart_add,
    CASE WHEN pe.visit_id IS NOT NULL THEN 1 ELSE 0 END AS purchase
  FROM product_page_events AS ppe
  LEFT JOIN purchase_events AS pe
    ON ppe.visit_id = pe.visit_id
)
```

Before we move on the results table below, I'd like to touch on the logic behind `abandoned` column in which `cart_add = 1` where a customer adds an item into the cart, but `purchase = 0` customer did not purchase and abandoned the cart.

```sql
SELECT 
  product_name, 
  product_category, 
  SUM(page_view) AS views,
  SUM(cart_add) AS cart_adds, 
  SUM(CASE WHEN cart_add = 1 AND purchase = 0 THEN 1 ELSE 0 END) AS abandoned,
  SUM(CASE WHEN cart_add = 1 AND purchase = 1 THEN 1 ELSE 0 END) AS purchases
FROM combined_table
GROUP BY product_id, product_name, product_category
ORDER BY product_id
```

<kbd><img width="845" alt="image" src="https://user-images.githubusercontent.com/81607668/136649917-ff1f7daa-9fb6-4077-9196-8596cd6eb424.png"></kbd>

**Solution for [Table 2]**
```sql
-- Run the `product_page_events`, `purchase_events`, and `combined_table` CTEs and query below concurrently
SELECT 
  product_category, 
  SUM(page_view) AS views,
  SUM(cart_add) AS cart_adds, 
  SUM(CASE WHEN cart_add = 1 AND purchase = 0 THEN 1 ELSE 0 END) AS abandoned,
  SUM(CASE WHEN cart_add = 1 AND purchase = 1 THEN 1 ELSE 0 END) AS purchases
FROM combined_table
GROUP BY product_category;
```

<kbd><img width="661" alt="image" src="https://user-images.githubusercontent.com/81607668/136650026-e6817dd2-ab30-4d5f-ab06-0b431f087dad.png"></kbd>


