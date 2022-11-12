**Schema (PostgreSQL v13)**

---
<p align=center><b>B. Data Analysis Questions</b>

---
**Query #1** How many customers has Foodie-Fi ever had?
    
    SELECT COUNT(DISTINCT(customer_id)) as total_customers
    FROM foodie_fi.subscriptions;

| total_customers |
| --------------- |
| 1000            |

---
**Query #2** What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value
    
    SELECT EXTRACT(MONTH FROM start_date) as months, COUNT(*)
    FROM foodie_fi.subscriptions
    WHERE plan_id = 0
    GROUP BY EXTRACT(MONTH FROM start_date)
    ORDER BY EXTRACT(MONTH FROM start_date);

| months | count |
| ------ | ----- |
| 1      | 88    |
| 2      | 68    |
| 3      | 94    |
| 4      | 81    |
| 5      | 88    |
| 6      | 79    |
| 7      | 89    |
| 8      | 88    |
| 9      | 87    |
| 10     | 79    |
| 11     | 75    |
| 12     | 84    |

---
**Query #3** What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name
    
    SELECT pl.plan_name, COUNT(*)
    FROM foodie_fi.plans pl
    JOIN foodie_fi.subscriptions sub
    ON pl.plan_id = sub.plan_id
    WHERE EXTRACT(year from sub.start_date) > 2020
    GROUP BY pl.plan_name
    ORDER BY COUNT(*) DESC;

| plan_name     | count |
| ------------- | ----- |
| churn         | 71    |
| pro annual    | 63    |
| pro monthly   | 60    |
| basic monthly | 8     |

---
**Query #4** What is the customer count and percentage of customers who have churned rounded to 1 decimal place?
    
    SELECT COUNT(DISTINCT(customer_id)) as churn_count,
    ROUND(COUNT(DISTINCT(customer_id))*100.0/(SELECT COUNT(DISTINCT(s.customer_id)) FROM foodie_fi.subscriptions s), 1) as chur_percent
    FROM foodie_fi.subscriptions
    WHERE plan_id = 4;

| churn_count | chur_percent |
| ----------- | ------------ |
| 307         | 30.7         |

---
**Query #5** How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?
    
    SELECT COUNT(tbl.customer_id) as total_count, 
    ROUND(COUNT(tbl.customer_id)*100.0/(SELECT COUNT(DISTINCT(s.customer_id)) FROM foodie_fi.subscriptions s)) as percentage 
       FROM (
    		SELECT customer_id, plan_id, LAG(plan_id,1,0)
    		OVER(PARTITION BY customer_id ORDER BY start_date)
    		FROM foodie_fi.subscriptions
      	) tbl
    WHERE tbl.plan_id = 4 AND tbl.lag = 0;

| total_count | percentage |
| ----------- | ---------- |
| 92          | 9          |

---
**Query #6** What is the number and percentage of customer plans after their initial free trial?
    
    SELECT 'Couldnt solve this';

| ?column?           |
| ------------------ |
| Couldnt solve this |

---
**Query #7** What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?
    
    SELECT pl.plan_name, COUNT(s.customer_id) 
    FROM foodie_fi.subscriptions s
    JOIN foodie_fi.plans pl
    ON s.plan_id = pl.plan_id
    WHERE s.start_date = '2020-12-31'
    GROUP BY pl.plan_name;

| plan_name | count |
| --------- | ----- |
| churn     | 1     |

---
**Query #8** How many customers have upgraded to an annual plan in 2020?
    
    SELECT COUNT(DISTINCT(customer_id)) as total_count
    FROM foodie_fi.subscriptions
    WHERE plan_id = 3
    AND EXTRACT(YEAR FROM start_date) = 2020;

| total_count |
| ----------- |
| 195         |

---
**Query #9** How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
    
    SELECT 'Couldnt solve this';

| ?column?           |
| ------------------ |
| Couldnt solve this |

---
**Query #10** Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)
    
    SELECT 'Couldnt solve this';

| ?column?           |
| ------------------ |
| Couldnt solve this |

---
**Query #11** How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
    
    SELECT 'Couldnt solve this';

| ?column?           |
| ------------------ |
| Couldnt solve this |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/rHJhRrXy5hbVBNJ6F6b9gJ/16)
