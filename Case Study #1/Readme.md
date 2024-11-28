# Case Study #1: Danny's Diner

![image](https://github.com/user-attachments/assets/759864ed-e62e-4906-a74c-54d9ebb8c610)

All information regarding this dataset exists [here](https://8weeksqlchallenge.com/case-study-1/)
# Introduction
Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

### Problem Statement
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

### Entity Relationship Diagram
![image](https://github.com/user-attachments/assets/8fb8cc08-3277-4c1b-b7cf-33926c61814f)


### Case Study Questions and Solutions 
**1. What is the total amount each customer spent at the restaurant**
 
````sql
    SELECT customer_id, SUM(price) AS total_price
    FROM sales
    JOIN menu
    USING(product_id)
    GROUP BY customer_id
    ORDER BY total_price DESC;
 ````  
##### Answer:
| customer_id | total_price |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

- Customer A spent a total of $76
- Customer B spent a total of $74
- Customer C spent a total of $36

**2. How many days has each customer visited the restaurant?**
````sql
SELECT
    customer_id, 
    COUNT(Distinct order_date) AS days_count
FROM sales
GROUP BY customer_id;
````

##### Answer:
| customer_id | days_count |
| ----------- | ---------- |
| A           | 4          |
| B           | 6          |
| C           | 2          |

- Customer A visited the restaurant 4 times
- Customer B visited 6 times
- Custoner C visited 2 times
 
**3.What was the first item from the menu purchased by each customer?**
````sql
 WITH fo AS(
    SELECT customer_id, product_name, order_date,
        DENSE_RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS rank
    FROM sales
    JOIN menu
    USING(product_id)
    )
    
    SELECT customer_id, product_name
    FROM fo
    WHERE rank = 1
    GROUP BY customer_id, product_name;
````
##### Answer:

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

- Customer A ordered two products on the same day, Curry and Sushi. It could be simultaneously or not, but both orders happened on the same day, which makes them both the first order.
- Customer B's first order was Curry, and
- Ramen for customer C. 

**4.What is the most purchased item on the menu and how many times was it purchased by all customers?**
````sql
SELECT 
        product_name, 
        COUNT(*) AS most_purchased_item
    FROM sales
    JOIN menu
    USING(product_id)
    GROUP BY product_name 
    ORDER BY most_purchased_item DESC
    LIMIT 1;
````
##### Answer:

| product_name | most_purchased_item |
| ------------ | ------------------- |
| ramen        | 8                   |

Ramen is the most purchased item on the menu, and it was purchased a total of 8 times.
Futher Analysis could focus on Ramen to uncover more insights.

**5.Which item was the most popular for each customer?**

````sql
 WITH max_order AS(
    SELECT 
        customer_id,
        product_name, 
        COUNT(*) AS item_count,
        DENSE_RANK() 
        OVER(PARTITION BY customer_id 
             ORDER BY COUNT(*) DESC) AS rank
    FROM sales
    JOIN menu
    USING(product_id)
    GROUP BY customer_id, product_name
     )
    SELECT customer_id, product_name, item_count
    FROM max_order
    WHERE rank = 1;
````
##### Answer:

| customer_id | product_name | item_count |
| ----------- | ------------ | ---------- |
| A           | ramen        | 3          |
| B           | ramen        | 2          |
| B           | curry        | 2          |
| B           | sushi        | 2          |
| C           | ramen        | 3          |

Ramen is the most popular item fot customer A and C, while customer B seems to like all the three items equally or is yet to decide. 

**6.Which item was purchased first by the customer after they became a member?**
````sql
WITH after_customer AS(
    SELECT customer_id, order_date, product_id, join_date,
        ROW_NUMBER() 
        OVER(PARTITION BY customer_id ORDER BY join_date) AS order_number
    FROM sales
    JOIN members
    USING(customer_id)
    WHERE order_date > join_date
    )
    
    SELECT customer_id, product_name
    FROM after_customer
    JOIN menu
    USING(product_id)
    WHERE order_number = 1
    ORDER BY customer_id;
````

##### Answer:

| customer_id | product_name |
| ----------- | ------------ |
| A           | ramen        |
| B           | sushi        |

Customer A purchased Ramen after they became a member
Customer B's first purchase was Sushi, and
Customer C is yet to take that big step (becoming a member)

**7.Which item was purchased just before the customer became a member?**
````sql
    WITH before_customer AS(
    SELECT customer_id, order_date, product_id, join_date,
        ROW_NUMBER() 
        OVER(PARTITION BY customer_id ORDER BY order_date DESC) AS order_number
    FROM sales
    JOIN members
    USING(customer_id)
    WHERE order_date < join_date
    )
    
    SELECT customer_id, product_name
    FROM before_customer
    JOIN menu
    USING(product_id)
    WHERE order_number = 1
    ORDER BY customer_id;
````
##### Answer:

| customer_id | product_name |
| ----------- | ------------ |
| A           | sushi        |
| B           | sushi        |

Both customers purchased Sushi before becoming a memeber

**8.What is the total items and amount spent for each member before they became a member?**
````sql
    SELECT s.customer_id, COUNT(*) AS total_items, SUM(price) AS total_amount
    FROM sales s
    JOIN menu m
    ON s.product_id = m.product_id
    JOIN members 
    ON s.customer_id = members.customer_id 
    AND order_date < join_date
    GROUP BY s.customer_id
    ORDER BY s.customer_id;
````
##### Answer:

| customer_id | total_items | total_amount |
| ----------- | ----------- | ------------ |
| A           | 2           | 25           |
| B           | 3           | 40           |

- Customer A spent a total of $25 dollar on 2 items
- Customer B spent a total of $40 dollar on 3 items

**9.If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**
````sql
    WITH customer_points AS(
    SELECT customer_id, 
        CASE WHEN product_name = 'sushi' THEN (price * 20)
        ELSE (price * 10) END AS points
    FROM sales
    JOIN menu
    USING (product_id)
    )
    SELECT customer_id, SUM(points) AS total_points
    FROM customer_points
    GROUP BY customer_id
    ORDER BY customer_id;
````
##### Answer:

| customer_id | total_points |
| ----------- | ------------ |
| A           | 860          |
| B           | 940          |
| C           | 360          |

- Customer A will have a total of 860 points
- Customer B will have a total of 940 points
- Customer C will have a total of 360 points  

**10.In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**
````sql
    WITH fw_CTE AS(
    SELECT customer_id, join_date, join_date + 6 AS first_week
    FROM members
    )
    
    SELECT sales.customer_id, 
    SUM(CASE
        WHEN order_date BETWEEN fw_CTE.join_date AND first_week THEN price *10 *2
         WHEN menu.product_name = 'sushi' THEN price * 10 *2
         ELSE price * 10 END) AS points
    FROM fw_CTE
    JOIN sales
    ON fw_CTE.customer_id = sales.customer_id
    JOIN menu 
    ON sales.product_id = menu.product_id
    AND order_date > join_date
    WHERE sales.order_date BETWEEN '2021-01-01' AND '2021-01-31'
    GROUP BY sales.customer_id
    ORDER BY sales.customer_id;
````
##### Answer:

| customer_id | points |
| ----------- | ------ |
| A           | 720    |
| B           | 320    |

- Total of 720 points for Customer A
- Total oof 320 points for Customer B

***

### Bonus Questions

**Join all the Things**
**Recreate the following table output using the available data.

output: customer_id	order_date	product_name	price	member**

````sql
    SELECT sales.customer_id, order_date, product_name, price,
    CASE
        WHEN order_date < join_date THEN 'N'
        WHEN order_date >= join_date THEN 'Y'
        ELSE 'N' END AS member
    FROM sales
    JOIN menu
    ON sales.product_id = menu.product_id 
    LEFT JOIN members
    ON members.customer_id = sales.customer_id
    ORDER BY sales.customer_id, order_date, product_name;
````
##### Answer:

| customer_id | order_date | product_name | price | member |
| ----------- | ---------- | ------------ | ----- | ------ |
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |

***
**Rank all the things**
**Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.**

##### Answer:
WITH membership AS(
    SELECT sales.customer_id, order_date, product_name, price,
    CASE
        WHEN order_date < join_date THEN 'N'
        WHEN order_date >= join_date THEN 'Y'
        ELSE 'N' END AS member  
    FROM sales
    JOIN menu
    ON sales.product_id = menu.product_id 
    LEFT JOIN members
    ON members.customer_id = sales.customer_id
    ORDER BY sales.customer_id, order_date, product_name
    )
    
    SELECT customer_id, order_date, product_name, price, member, 
    CASE 
    WHEN member = 'Y' THEN 
    DENSE_RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date)
    ELSE NULL
    END AS ranking
    FROM membership;

| customer_id | order_date | product_name | price | member | ranking |
| ----------- | ---------- | ------------ | ----- | ------ | ------- |
| A           | 2021-01-01 | curry        | 15    | N      | Null    |
| A           | 2021-01-01 | sushi        | 10    | N      | Null    |
| A           | 2021-01-07 | curry        | 15    | Y      | 1       |
| A           | 2021-01-10 | ramen        | 12    | Y      | 2       |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| B           | 2021-01-01 | curry        | 15    | N      | Null    |
| B           | 2021-01-02 | curry        | 15    | N      | Null    |
| B           | 2021-01-04 | sushi        | 10    | N      | Null    |
| B           | 2021-01-11 | sushi        | 10    | Y      | 1       |
| B           | 2021-01-16 | ramen        | 12    | Y      | 2       |
| B           | 2021-02-01 | ramen        | 12    | Y      | 3       |
| C           | 2021-01-01 | ramen        | 12    | N      | Null    |
| C           | 2021-01-01 | ramen        | 12    | N      | Null    |
| C           | 2021-01-07 | ramen        | 12    | N      | Null    |

***

#### Insights
1. Customer A and Customer B have the highest total spending, at $76 and $74, respectively. In contrast, Customer C's total spend is significantly low at $36.
2. Customer B visited the restaurant the most, with a total of 6 visits, Customer A visited 4 times, and customer C visited only twice.
3. Ramen is the most ordered item on the menu with a total of 8 purchases across all customers. It is also the favorite item for both Customer A and Customer C indicating its wide appeal.

***

#### Recommendations
1. Conduct a survey, promotional email or campaign targeting Customer C to understand their limited visits and spending habits. Encourage repeat visits from customer C with targeted loyalty rewards or discounts.
2. Ensure Ramen is consistently well-stocked to meet customer demand, given its popularity among customers.
3. Offer special discounts, promotions or incentives for frequent visitors to strengthen loyalty and encourage continued patronage.
  
