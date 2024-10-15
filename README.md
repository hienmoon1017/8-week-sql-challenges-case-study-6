# 8 Week SQL Challenges by Danny | Case Study #6 - Clique Bait
### ERD for Database

![image](https://github.com/user-attachments/assets/bace4774-1841-4cf8-9d22-cb5c925a1e89)


## Result of Questions
### 1. Data Cleansing Steps
 ```sql
-- Convert product_id of page_hierarchy to integer
UPDATE clique_bait.page_hierarchy
SET product_id = case when product_id = 'null' then null
						else product_id
						end;

ALTER TABLE clique_bait.page_hierarchy
ADD COLUMN cl_product_id integer;

UPDATE clique_bait.page_hierarchy
SET cl_product_id = product_id :: integer;
```
### 2. Digital Analysis
#### 2.1 How many users are there?
```sql
SELECT count(distinct user_id) as users_count
FROM clique_bait.users;
```

_Result:_

![image](https://github.com/user-attachments/assets/beb2ab09-c69e-4d2c-9f8e-3ae74ec230a3)

#### 2.2 How many cookies does each user have on average?
```sql
SELECT count(cookie_id) / count(distinct user_id) as avg_cookie_per_user
FROM clique_bait.users;
```

_Result:_

![image](https://github.com/user-attachments/assets/4517c3f3-3810-4a45-a10e-040a71322881)

#### 2.3 What is the unique number of visits by all users per month?
```sql
SELECT to_char(event_time,'MM-YYYY') as month
,count(distinct visit_id) as unique_visits
FROM clique_bait.events 
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/99c81b90-5e26-4b0f-a85a-bedd706b1500)


#### 2.4 What is the number of events for each event type?
```sql
SELECT event_type
,count(*) as event_count
FROM clique_bait.events
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/e9611bcd-eff0-4ccd-b39d-4ba5c7a0a70b)

#### 2.5 What is the percentage of visits which have a purchase event?
```sql
-- '3' is purchase event
SELECT count(distinct visit_id) filter (where event_type = '3') *100 / count(distinct visit_id) || '%' as percentage_purchase_visits
FROM clique_bait.events;
```

_Result:_

![image](https://github.com/user-attachments/assets/d2575c70-cbf7-46df-86a6-8dbb7cdf9cf9)


#### 2.6 What is the percentage of visits which view the checkout page but do not have a purchase event?
```sql
WITH conditions as
(
	SELECT e.visit_id
	,max(case when ph.page_id = '12' then 1 else 0 end) as visit_has_checkout -- 12 is checkout page
	,max(case when e.event_type = '3' then 1 else 0 end) as visit_has_purchase -- 3 is purchase
	FROM clique_bait.events e
	JOIN clique_bait.page_hierarchy ph on ph.page_id = e.page_id
	GROUP BY 1
)
SELECT count(distinct visit_id) filter (where visit_has_checkout = 1 and visit_has_purchase = 0) *100 / count(distinct visit_id) || '%' as percentage_checkout_not_purchase
FROM conditions;
```

_Result:_

![image](https://github.com/user-attachments/assets/0bed82d7-d2a6-45d3-b509-ad9223511895)

#### 2.7 What are the top 3 pages by number of views?
```sql
SELECT page_id
,count(case when event_type = '1' then 1 end) as views_count
FROM clique_bait.events
GROUP BY 1
ORDER BY 2 DESC
LIMIT 3;
```

_Result:_

![image](https://github.com/user-attachments/assets/8d953e32-0ca7-44a0-8e60-2c40952f244a)

#### 2.8 What is the number of views and cart adds for each product category?
```sql
SELECT ph.product_category
,count(case when e.event_type = '1' then 1 end) as views_count
,count(case when e.event_type = '2' then 1 end) as add_to_cart_count
FROM clique_bait.page_hierarchy ph
JOIN clique_bait.events e on e.page_id = ph.page_id
GROUP BY 1
ORDER BY 2 DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/7cfad397-f492-4373-89df-fcf05519343d)

#### 2.9 What are the top 3 products by purchases?
```sql
SELECT ph.product_id
,count(case when e.event_type = '3' then 1 end) as purchase_count
FROM clique_bait.page_hierarchy ph
JOIN clique_bait.events e on e.page_id = ph.page_id
GROUP BY 1
ORDER BY 2 DESC
LIMIT 3;
```

_Result:_

![image](https://github.com/user-attachments/assets/26fb51db-06d2-4fc8-9a39-bb0bbd4658b9)

### 3. Product Funnel Analysis
#### 3.1 for individual product
```sql 
WITH product_events as
(
	SELECT ph.cl_product_id
	,count(case when e.event_type = '1' then 1 end) as viewed_count
	,count(case when e.event_type = '2' then 1 end) as added_to_cart_count	
	,count(case when e.event_type = '3' then 1 end) as purchased_count
	FROM clique_bait.events e
	JOIN clique_bait.page_hierarchy ph on ph.page_id = e.page_id
	WHERE ph.product_id is not null
	GROUP BY 1
)
SELECT pe.cl_product_id as product_id
,pe.viewed_count
,pe.added_to_cart_count
,pe.purchased_count
,coalesce(pe.added_to_cart_count - count(distinct case when e.event_type = '3' then e.page_id end),0) as abandoned_count
,pe.added_to_cart_count *100 / nullif(pe.viewed_count,0) || '%' as conversion_rate_from_view_to_cart
,pe.purchased_count *100 / nullif(pe.added_to_cart_count,0) || '%' as conversion_rate_from_cart_to_purchase
FROM product_events pe
LEFT JOIN clique_bait.events e on e.page_id in (
													SELECT ph.page_id
													FROM clique_bait.page_hierarchy ph
													WHERE ph.cl_product_id = pe.cl_product_id
												)
GROUP BY 1,2,3,4
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/40612742-6003-4d34-801e-2ac2692a5c21)


#### 3.2 for product category
```sql
WITH product_cate_events as
(
	SELECT ph.product_category
	,count(case when e.event_type = '1' then 1 end) as viewed_count
	,count(case when e.event_type = '2' then 1 end) as added_to_cart_count
	,count(case when e.event_type = '3' then 1 end) as purchased_count
	FROM clique_bait.events e
	JOIN clique_bait.page_hierarchy ph on ph.page_id = e.page_id
	WHERE ph.product_category is not null
	GROUP BY 1
)
SELECT pce.product_category
,pce.viewed_count
,pce.added_to_cart_count
,pce.purchased_count
,coalesce(pce.added_to_cart_count - count(case when e.event_type = '3' then e.page_id end),0) as abandoned_count
,pce.added_to_cart_count *100 / nullif(pce.viewed_count,0) || '%' as conversion_rate_from_view_to_cart
,pce.purchased_count *100 / nullif(pce.added_to_cart_count,0) || '%' as conversion_rate_from_cart_to_purchase
FROM product_cate_events pce
LEFT JOIN clique_bait.events e on e.page_id in
										(
											SELECT ph.page_id
											FROM clique_bait.page_hierarchy ph
											WHERE ph.product_category = pce.product_category
										)
GROUP BY 1,2,3,4
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/cd7d0c77-ef75-4763-9de4-9f446b9d4b5d)


### 4. Campaigns Analysis
```sql
WITH visit_metrics as
(
	SELECT u.user_id
	,e.visit_id	
	,min(date(e.event_time)) as visit_start_time
	,count(case when e.event_type ='1' then e.page_id end) as page_views
	,count(case when e.event_type = '2' then 1 end) as cart_adds
	,max(case when e.event_type = '3' then 1 else 0 end) as purchase
	,string_agg(case when e.event_type = '2' then ph.product_category end,',' order by e.sequence_number) as cart_products -- Optional column
	FROM clique_bait.users u
	JOIN clique_bait.events e on u.cookie_id = e.cookie_id
	JOIN clique_bait.page_hierarchy ph on ph.page_id = e.page_id
	GROUP BY 1,2
)
,campaign_mapping as
(
	SELECT ci.campaign_name
	,vm.user_id
	,vm.visit_id
	,vm.visit_start_time
	,vm.page_views
	,vm.cart_adds
	,vm.purchase
	,vm.cart_products
	FROM visit_metrics vm
	JOIN clique_bait.campaign_identifier ci on vm.visit_start_time between date(ci.start_date) and date(ci.end_date)
)
,final_metrics as
(
	SELECT cm.user_id
	,cm.visit_id
	,cm.visit_start_time
	,cm.page_views
	,cm.cart_adds
	,cm.purchase
	,cm.campaign_name
	,cm.cart_products
	,count(case when e.event_type ='4' then 1 end) as ad_impression
	,count(case when e.event_type ='5' then 1 end) as ad_click
	FROM campaign_mapping cm
	JOIN clique_bait.events e on e.visit_id = cm.visit_id
	GROUP BY 1,2,3,4,5,6,7,8
)
SELECT *
FROM final_metrics
ORDER BY 1,2;
```

_Result:_

![image](https://github.com/user-attachments/assets/7234d053-deb6-496e-89ee-3d5b02694eae)


Thank you for stopping by, and I'm pleased to connect with you, my new friend!

Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.

Wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
