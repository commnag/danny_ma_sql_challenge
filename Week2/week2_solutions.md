**Schema (PostgreSQL v13)**

---

**Cleaning the data** 

    UPDATE pizza_runner.customer_orders
    SET exclusions = NULL
    WHERE exclusions = 'null' 
    OR exclusions = '';

    UPDATE pizza_runner.customer_orders
    SET extras = NULL
    WHERE extras = 'null' 
    OR extras = '';

    UPDATE pizza_runner.runner_orders
    SET pickup_time = NULL
    WHERE pickup_time = 'null' 
    OR pickup_time = '';

    UPDATE pizza_runner.runner_orders
    SET distance = NULL
    WHERE distance = 'null' 
    OR distance = '';

    UPDATE pizza_runner.runner_orders
    SET duration = NULL
    WHERE duration = 'null' 
    OR duration = '';

    UPDATE pizza_runner.runner_orders
    SET cancellation = NULL
    WHERE cancellation NOT LIKE '%Cancellation';

---
<p align=center><b>A. Pizza Metrics</b>

---
**Query #1** How many pizzas were ordered?

    SELECT COUNT(*) as total_ordered
    FROM pizza_runner.customer_orders;

| total_ordered |
| ------------- |
| 14            |

---
**Query #2** How many unique customer orders were made?

    SELECT COUNT(DISTINCT(order_id)) as total_ordered
    FROM pizza_runner.customer_orders;

| total_ordered |
| ------------- |
| 10            |

---
**Query #3** How many successful orders were delivered by each runner?

    SELECT COUNT(*) as total_delivered
    FROM pizza_runner.runner_orders
    WHERE cancellation IS NULL;

| total_delivered |
| --------------- |
| 8               |

---
**Query #4** How many of each type of pizza was delivered?

    SELECT pizza_name, COUNT(*) as total_delivered
    FROM pizza_runner.pizza_names na
    INNER JOIN pizza_runner.customer_orders co
    ON na.pizza_id = co.pizza_id
    INNER JOIN pizza_runner.runner_orders ro
    ON ro.order_id = co.order_id
    WHERE ro.cancellation IS NULL
    GROUP BY na.pizza_name;

| pizza_name | total_delivered |
| ---------- | --------------- |
| Meatlovers | 9               |
| Vegetarian | 3               |

---
**Query #5** How many Vegetarian and Meatlovers were ordered by each customer?

    SELECT co.customer_id, na.pizza_name, COUNT(*) as total_ordered
    FROM pizza_runner.customer_orders co
    INNER JOIN pizza_runner.pizza_names na
    ON na.pizza_id = co.pizza_id
    GROUP BY co.customer_id, na.pizza_name
    ORDER BY co.customer_id, na.pizza_name;

| customer_id | pizza_name | total_ordered |
| ----------- | ---------- | ------------- |
| 101         | Meatlovers | 2             |
| 101         | Vegetarian | 1             |
| 102         | Meatlovers | 2             |
| 102         | Vegetarian | 1             |
| 103         | Meatlovers | 3             |
| 103         | Vegetarian | 1             |
| 104         | Meatlovers | 3             |
| 105         | Vegetarian | 1             |

---
**Query #6** What was the maximum number of pizzas delivered in a single order?

    SELECT COUNT(*) as max_pizza_delivered
    FROM pizza_runner.customer_orders co
    INNER JOIN pizza_runner.runner_orders ro
    ON ro.order_id = co.order_id
    WHERE ro.cancellation IS NULL
    GROUP BY co.order_id
    ORDER BY COUNT(*) DESC
    LIMIT 1;

| max_pizza_delivered |
| ------------------- |
| 3                   |

---
**Query #8** How many pizzas were delivered that had both exclusions and extras?

    SELECT COUNT(co.pizza_id) 
    FROM pizza_runner.customer_orders co
    JOIN pizza_runner.runner_orders ro
    ON co.order_id = ro.order_id
    WHERE co.exclusions IS NOT NULL
    AND co.extras IS NOT NULL
    AND ro.cancellation IS NULL;

| count |
| ----- |
| 1     |

---
**Query #9** What was the total volume of pizzas ordered for each hour of the day?

    SELECT EXTRACT(HOUR FROM order_time) as hours, COUNT(pizza_id) as total_volume_of_pizzas 
    FROM pizza_runner.customer_orders
    GROUP BY EXTRACT(HOUR FROM order_time)
    ORDER BY EXTRACT(HOUR FROM order_time);

| hours | total_volume_of_pizzas |
| ----- | ---------------------- |
| 11    | 1                      |
| 13    | 3                      |
| 18    | 3                      |
| 19    | 1                      |
| 21    | 3                      |
| 23    | 3                      |

---
<p align=center><b>B. Runner and Customer Experience</b>

---
**Query #1** How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

    SELECT EXTRACT('week' FROM registration_date+3) as week, COUNT(runner_id) 
    FROM pizza_runner.runners
    GROUP BY EXTRACT('week' FROM registration_date+3)
    ORDER BY EXTRACT('week' FROM registration_date+3);

| week | count |
| ---- | ----- |
| 1    | 2     |
| 2    | 1     |
| 3    | 1     |

---
**Query #4** What was the average distance travelled for each customer?

    SELECT co.customer_id, ROUND(AVG(substring(ro.distance from '\d*')::INTEGER), 1) as avg_dist_travelled
    FROM pizza_runner.runner_orders ro
    JOIN pizza_runner.customer_orders co
    ON ro.order_id = co.order_id
    GROUP BY co.customer_id
    ORDER BY co.customer_id;

| customer_id | avg_dist_travelled |
| ----------- | ------------------ |
| 101         | 20.0               |
| 102         | 16.3               |
| 103         | 23.0               |
| 104         | 10.0               |
| 105         | 25.0               |

---
**Query #5** What was the difference between the longest and shortest delivery times for all orders?

    SELECT (MAX(substring(duration from '\d*')::INTEGER) - MIN(substring(duration from '\d*')::INTEGER)) as duration_diff
    FROM pizza_runner.runner_orders;

| duration_diff |
| ------------- |
| 30            |

---
**Query #6** What was the average speed for each runner for each delivery and do you notice any trend for these values?

    SELECT runner_id, order_id, ROUND((substring(distance from '\d*')::INTEGER * 60.0)/(substring(duration from '\d*')::INTEGER), 1) as speed_kmph
    FROM pizza_runner.runner_orders
    WHERE cancellation IS NULL
    GROUP BY runner_id, order_id, distance, duration
    ORDER BY runner_id, order_id;

| runner_id | order_id | speed_kmph |
| --------- | -------- | ---------- |
| 1         | 1        | 37.5       |
| 1         | 2        | 44.4       |
| 1         | 3        | 39.0       |
| 1         | 10       | 60.0       |
| 2         | 4        | 34.5       |
| 2         | 7        | 60.0       |
| 2         | 8        | 92.0       |
| 3         | 5        | 40.0       |

---
**Query #7** What is the successful delivery percentage for each runner?

    SELECT r1.runner_id, ROUND(COUNT(r1.order_id)*100.0/(SELECT COUNT(*) FROM pizza_runner.runner_orders WHERE runner_id = r1.runner_id)) as delivery_percentage
    FROM pizza_runner.runner_orders r1
    WHERE r1.cancellation IS NULL
    GROUP BY r1.runner_id
    ORDER BY COUNT(r1.order_id) > 0;

| runner_id | delivery_percentage |
| --------- | ------------------- |
| 1         | 100                 |
| 2         | 75                  |
| 3         | 50                  |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/7VcQKQwsS3CTkGRFG7vu98/65)
