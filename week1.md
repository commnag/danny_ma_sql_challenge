**Schema (PostgreSQL v15 (Beta))**

    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

---

**Query #1**

    SELECT sa.customer_id, SUM(me.price) as Total_Spend
    FROM dannys_diner.sales sa
    INNER JOIN dannys_diner.menu me
    ON sa.product_id = me.product_id
    GROUP BY sa.customer_id;

| customer_id | total_spend |
| ----------- | ----------- |
| B           | 74          |
| C           | 36          |
| A           | 76          |

---
**Query #2**

    SELECT customer_id, count(distinct(order_date)) as Total_Visits
    FROM dannys_diner.sales
    GROUP BY customer_id;

| customer_id | total_visits |
| ----------- | ------------ |
| A           | 4            |
| B           | 6            |
| C           | 2            |

---
**Query #3**

    SELECT tbl.customer_id, tbl.product_id
    FROM
    	(SELECT *, RANK()
    	OVER(PARTITION BY customer_id ORDER BY order_date)
    	FROM dannys_diner.sales) AS tbl
    WHERE tbl.rank=1;

| customer_id | product_id |
| ----------- | ---------- |
| A           | 1          |
| A           | 2          |
| B           | 2          |
| C           | 3          |
| C           | 3          |

---
**Query #4**

    SELECT customer_id, count(product_id)
    FROM dannys_diner.sales
    WHERE product_id = (
      	SELECT product_id 
    	FROM dannys_diner.sales
    	GROUP BY product_id
    	ORDER BY count(product_id) DESC
    	LIMIT 1
      )
    GROUP BY customer_id;

| customer_id | count |
| ----------- | ----- |
| B           | 2     |
| A           | 3     |
| C           | 3     |

---
**Query #5**

    SELECT tbl2.customer_id, tbl2.product_id
    FROM
    	(SELECT tbl.*, RANK()
    	OVER(PARTITION BY tbl.customer_id ORDER BY tbl.cnt DESC)
    	FROM (
    		SELECT customer_id, product_id, count(product_id) as cnt
    		FROM dannys_diner.sales
    		GROUP BY customer_id, product_id
    		ORDER BY customer_id, count(product_id) DESC
      	) as tbl
     	) as tbl2
    WHERE tbl2.rank=1;

| customer_id | product_id |
| ----------- | ---------- |
| A           | 3          |
| B           | 3          |
| B           | 1          |
| B           | 2          |
| C           | 3          |

---
**Query #6**

    SELECT tbl2.customer_id, tbl2.product_id
    FROM (
    	SELECT *, RANK()
    	OVER(PARTITION BY tbl.customer_id ORDER BY tbl.order_date)
    	FROM (
    		SELECT sa.customer_id, sa.product_id, sa.order_date, me.join_date
    		FROM dannys_diner.sales sa
    		INNER JOIN dannys_diner.members me
    		ON sa.customer_id = me.customer_id
    		AND sa.order_date>me.join_date
      	) as tbl
      ) as tbl2
    WHERE tbl2.rank=1;

| customer_id | product_id |
| ----------- | ---------- |
| A           | 3          |
| B           | 1          |

---
**Query #7**

    SELECT tbl2.customer_id, tbl2.product_id
    FROM (
    	SELECT *, RANK()
    	OVER(PARTITION BY tbl.customer_id ORDER BY tbl.order_date DESC)
    	FROM (
    		SELECT sa.customer_id, sa.product_id, sa.order_date, me.join_date
    		FROM dannys_diner.sales sa
    		INNER JOIN dannys_diner.members me
    		ON sa.customer_id = me.customer_id
    		AND sa.order_date<=me.join_date
      	) as tbl
      ) as tbl2
    WHERE tbl2.rank=1;

| customer_id | product_id |
| ----------- | ---------- |
| A           | 2          |
| B           | 1          |

---
**Query #8**

    SELECT sa.customer_id, COUNT(sa.product_id), SUM(men.price) 
    FROM dannys_diner.sales sa
    INNER JOIN dannys_diner.members me
    ON sa.customer_id = me.customer_id
    INNER JOIN dannys_diner.menu men
    ON men.product_id = sa.product_id
    AND sa.order_date < me.join_date
    GROUP BY sa.customer_id;

| customer_id | count | sum |
| ----------- | ----- | --- |
| B           | 3     | 40  |
| A           | 2     | 25  |

---
**Query #9**

    SELECT tbl.customer_id, SUM(tbl.points) 
    FROM (
    	SELECT sa.customer_id, men.product_name, men.price,
    	CASE
    		WHEN men.product_name = 'sushi' THEN men.price*2
        	ELSE men.price
    	END as points
    	FROM dannys_diner.sales sa
    	INNER JOIN dannys_diner.menu men
    	ON sa.product_id = men.product_id
    	) AS tbl
    GROUP BY tbl.customer_id;

| customer_id | sum |
| ----------- | --- |
| B           | 94  |
| C           | 36  |
| A           | 86  |

---
**Query #10**

    SELECT tbl.customer_id, SUM(tbl.points) 
    FROM 
    	(SELECT sa.customer_id, sa.order_date, men.product_id, 
    	CASE
    		WHEN me.join_date + 7 >= sa.order_date THEN men.price*2
       	 	ELSE men.price
    	END as points
    	FROM dannys_diner.sales sa
    	INNER JOIN dannys_diner.members me
    	ON sa.customer_id = me.customer_id
    	AND sa.order_date >= me.join_date
    	INNER JOIN dannys_diner.menu men
    	ON men.product_id = sa.product_id
    	WHERE date_part('month', order_date)=1
    	) as tbl
    GROUP BY tbl.customer_id;

| customer_id | sum |
| ----------- | --- |
| B           | 44  |
| A           | 102 |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138)
