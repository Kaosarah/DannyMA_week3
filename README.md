# DannyMA_week3

///
--QUESTION ONE;


SELECT COUNT(DISTINCT customer_id) customers FROM foodie_fi.subscriptions;

SELECT a.plan_name, b.customer_id,b. start_date FROM foodie_fi.plans a
    LEFT JOIN foodie_fi.subscriptions b ON a.plan_id = b.plan_id
ORDER BY b.customer_id, b.start_date;


--QUESTION (2)

SELECT COUNT(*),DATE_TRUNC('month', b.start_date) mon FROM foodie_fi.plans a
    LEFT JOIN foodie_fi.subscriptions b ON a.plan_id = b.plan_id
WHERE a.plan_name = 'trial'
GROUP BY mon
ORDER BY mon;

--Q(3)

SELECT a.plan_name, COUNT(a.plan_id) FROM foodie_fi.plans a
    LEFT JOIN foodie_fi.subscriptions b ON a.plan_id = b.plan_id
WHERE  b.start_date  > '2020-12-31'
GROUP BY a.plan_name;

--Q(4) Number of customers that churned and percentage churn

SELECT COUNT( b.customer_id) FROM foodie_fi.plans a
LEFT JOIN foodie_fi.subscriptions b ON a.plan_id = b.plan_id
WHERE a.plan_name = 'churn';

SELECT churn_count, ROUND((churn_count/total_count  :: NUMERIC * 100), 2)
FROM
(SELECT SUM(CASE WHEN plan_id = 4 THEN 1 END)  churn_count,
    COUNT(DISTINCT customer_id) total_count
    FROM foodie_fi.subscriptions) a;

--Q (checking duplicate)
SELECT * FROM
(SELECT *, row_number() OVER(PARTITION BY customer_id, plan_id, start_date ORDER BY customer_id ) ROW
FROM foodie_fi.subscriptions) a
WHERE ROW > 1;

--Q(5) How many customers have churned straight after their initial free trial 
-- what percentage is this rounded to the nearest whole number?
SELECT CHURNED, ROUND(CHURNED/TOTAL_CUS:: NUMERIC * 100) percent_churned_aftertrial 
FROM (SELECT COUNT (CASE WHEN plan_id = 4 and row_num = 2 THEN 1 END) CHURNED,
 COUNT(DISTINCT customer_id) TOTAL_CUS
FROM(
 SELECT *, 
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) AS row_num
    FROM foodie_fi.subscriptions) a) b;  ---92 customers churned after trials


--Other method to solve question 5

SELECT churned_customers,
total_customers,
ROUND(churned_customers/total_customers::FLOAT*100) AS percent_churned_after_trial
FROM (
  SELECT 
  COUNT(CASE WHEN plan_id = 0 AND lead_plan = 4 THEN 1 END) AS churned_customers,
  COUNT(DISTINCT customer_id) AS total_customers
  FROM (
    SELECT *,
    LEAD(plan_id) OVER (PARTITION BY customer_id ORDER BY start_date) AS lead_plan
    FROM foodie_fi.subscriptions
    ORDER BY customer_id
  ) AS tmp
) AS tmp;

-- Q(6) What is the number and percentage of customer plans after their initial free trial?

SELECT customer, lead, ROUND(customer/ sum(customer) OVER() * 100, 2)
FROM(
SELECT COUNT(customer_id :: NUMERIC) customer , lead
FROM(
    SELECT *, LEAD(plan_id,1) OVER(PARTITION BY customer_id ORDER BY start_date) lead
 FROM foodie_fi.subscriptions) a
WHERE plan_id = 0
GROUP BY lead) b
GROUP BY lead, customer;

--Q7 What is the customer count and percentage breakdown 
--of all 5 plan_name values at 2020-12-31?

WITH row_num AS (
   SELECT *, ROW_number() OVER( PARTITION BY customer_id ORDER BY start_date) num
   FROM foodie_fi.subscriptions
   WHERE start_date <= '2020-12-31'
),
customer AS(
SELECT customer_id, MAX(num) maximum_row
    FROM row_num
      GROUP BY customer_id
        ORDER BY customer_id
)
SELECT Customer_No, ROUND(Customer_No/ SUM(Customer_No) OVER() * 100, 2) Percent_of_customer, plan_id
  FROM(
SELECT COUNT(customer_id :: NUMERIC) Customer_No, plan_id
FROM (
SELECT a.customer_id, maximum_row, b.plan_id
FROM customer a
 INNER JOIN row_num b
ON a.customer_id = b.customer_id AND a.maximum_row = b.num) c
GROUP BY plan_id
  ) d
GROUP BY Customer_No, plan_id;

--Q8 How many customers have upgraded to an annual plan in 2020?
WITH start_year AS (
    SELECT *, date_part('year',start_date) yr
      FROM foodie_fi.subscriptions
)
SELECT COUNT(DISTINCT customer_id), 
      plan_id
      FROM start_year
        WHERE yr = '2020' AND plan_id = 3
          GROUP BY plan_id;

--- Q8 Other method

SELECT COUNT (DISTINCT customer_id)
    FROM foodie_fi.subscriptions
WHERE plan_id = 3 AND EXTRACT(year FROM start_date) = 2020;

---Q9 
-- How many days on average does it take for a customer to an annual plan 
--from the day they join Foodie-Fi?

WITH annual_plan AS(
SELECT *
   FROM foodie_fi.subscriptions
WHERE plan_id = 3),
trial AS(
  SELECT *
      FROM foodie_fi.subscriptions
        WHERE plan_id = 0
)
SELECT ROUND(AVG(a.start_date  - b.start_date),0) Average_days
FROM annual_plan a
LEFT JOIN trial b
ON a.customer_id = b.customer_id;

--Q9 Other method

-- B.9
SELECT ROUND(AVG(annual.start_date - trial.start_date),0) AS avg_days
FROM foodie_fi.subscriptions AS trial
JOIN foodie_fi.subscriptions AS annual USING (customer_id)
WHERE trial.plan_id = 0 AND annual.plan_id = 3;


---Q 10 Can you further breakdown this average value into 30 day
  -- periods (i.e. 0-30 days, 31-60 days etc)
WITH section AS(
SELECT *, CASE WHEN days_dif BETWEEN 0 AND 30 THEN '0-30'
       WHEN days_dif BETWEEN 31 AND 60 THEN '31-60'
       WHEN days_dif BETWEEN 61 AND 90 THEN '61-90'
       WHEN days_dif BETWEEN 91 AND 120 THEN '91-120' 
       WHEN days_dif BETWEEN 121 AND 150 THEN '121-150'
       WHEN days_dif BETWEEN 151 AND 180 THEN '151- 180' 
       WHEN days_dif > 180 THEN '180+' END section
FROM
  (
  SELECT annual.customer_id,
    (annual.start_date - trial.start_date) AS days_dif
      FROM foodie_fi.subscriptions AS trial
        JOIN foodie_fi.subscriptions AS annual USING (customer_id)
          WHERE trial.plan_id = 0 AND annual.plan_id = 3
              ORDER BY days_dif
  ) a
  ),
  avg AS (
    SELECT ROUND(AVG(days_dif),0) average, section
        FROM section
            GROUP BY section
  ),
 customer AS (
    SELECT COUNT(DISTINCT customer_id) customer,
           section
            FROM section
                GROUP BY section
  ) 
  SELECT b.customer No_customer, a.average, b.section
  FROM avg a
  JOIN customer b
  USING (section)
  ORDER BY b.customer;
  

WITH customer_upgrade AS (
SELECT trial.customer_id, 
  trial.start_date AS trial, 
  annual.start_date AS annual,
  annual.start_date - trial.start_date AS date_interval
FROM foodie_fi.subscriptions AS trial
JOIN foodie_fi.subscriptions AS annual USING (customer_id)
  WHERE trial.plan_id = 0 AND annual.plan_id = 3
),
SC AS(
  SELECT *,
  CASE WHEN date_interval BETWEEN 0 AND 30 THEN '1 0-30'
      WHEN date_interval BETWEEN 31 AND 60 THEN '2 31-60'
      WHEN date_interval BETWEEN 61 AND 90 THEN '3 61-90'
      WHEN date_interval BETWEEN 91 AND 120 THEN '4 91-120'
      WHEN date_interval BETWEEN 121 AND 150 THEN '5 121-150'
      WHEN date_interval BETWEEN 151 AND 180 THEN '6 151-180'
      WHEN date_interval > 180 THEN '7 181+' END AS interval_group
  FROM customer_upgrade
) 
SELECT SPLIT_PART(interval_group, ' ',2) AS interval_days, 
ROUND(AVG(date_interval),1) AS avg_days,
COUNT(*) AS no_of_customers
FROM SC
GROUP BY interval_group
ORDER BY interval_group;


--Q 11 How many customers downgraded from a pro monthly to a basic monthly plan in 2020?
 SELECT customer_id, plan_id, lead_plan
 FROM(
 SELECT *,
    LEAD(plan_id) OVER (PARTITION BY customer_id ORDER BY start_date) AS lead_plan,
    EXTRACT(YEAR FROM start_date) AS yr
    FROM foodie_fi.subscriptions
    ORDER BY customer_id, start_date) a
WHERE lead_plan = 1 AND plan_id = 2 AND yr = '2020';


-- Q 11 (2nd method)
WITH customer_downgrade AS (
SELECT pro.customer_id, 
  pro.start_date AS pro_date,
  pro.plan_id,
  basic.start_date AS basic_date,
  basic.plan_id
FROM foodie_fi.subscriptions AS pro
JOIN foodie_fi.subscriptions AS basic USING (customer_id)
  WHERE pro.plan_id = 2 AND basic.plan_id = 1 
  AND pro.start_date < basic.start_date
  AND EXTRACT(year FROM basic.start_date) = 2020
)
SELECT * FROM customer_downgrade;

-- B.11 (3rd method)
WITH base_table AS (
  SELECT customer_id, STRING_AGG(plan_id::TEXT, ', ') AS customer_journey
  FROM foodie_fi.subscriptions
  WHERE EXTRACT(year FROM start_date) = 2020
  GROUP BY customer_id
)
SELECT * FROM base_table
WHERE customer_journey LIKE '%2, 1%';



 ----SECTION 2 

 SELECT customer_id, 
        p.plan_name,
        s.plan_id,
        start_date
    FROM foodie_fi.plans p
    JOIN foodie_fi.subscriptions s
        ON p.plan_id = s.plan_id
        WHERE s.plan_id = 0 AND s.plan_id = 4
            ORDER BY customer_id, s.plan_id
  






--C answer alternative (in progress)

WITH customers AS (
            SELECT customer_id, 
              plan_id, 
              start_date
  			,plan_name
  			,price
  			,customer_id:: text || start_date:: text as pkey
            FROM foodie_fi.subscriptions
  	       left join foodie_fi.plans using (plan_id)
            WHERE plan_id <> 0 and EXTRACT(year from start_date) <> 2021
            ORDER BY 1,2
           
),

-- This code fetch all customer without plan id 4
cust2 as (select * from customers where plan_id <> 4),

-- This code create an array aggregrate from planid
x0 as (select customer_id,ARRAY_AGG(plan_id) AS plan_journey  
		from customers
group by 1
order by 1),

 -- This code focus on on customers with single plan journey of 1 or 2 
plan1 as (
            select customer_id,plan_journey from x0
            where plan_journey = array[1]  or plan_journey = array[2]),
 -- This code focus  on customers with plan journey 1,2
plan1or2 as (
            select customer_id,plan_journey from x0
            where  plan_journey = array[1,2]),
 -- This code focus on on customers with plan journey 1,2,3            
plan123 as (select customer_id,plan_journey from x0
            where plan_journey = array[1,2,3] or plan_journey = array[1,3]
            or plan_journey = array[2,3] or plan_journey = array[3]),
 -- This code focus on customers with plan journey 1,2,3,4             
plan1234 as(select customer_id,plan_journey from x0
            where plan_journey = array[1,2,3,4] or plan_journey = array[1,4]
            or plan_journey = array[2,4] or plan_journey = array[3,4] 
             or plan_journey = array[4]  or plan_journey = array[1,2,4] 
              or plan_journey = array[1,3,4]),
              
-- This code generate date for customer with a single plan journey of 1 or 2
p1 as (select customer_id 
       ,GENERATE_SERIES(MIN(c.start_date)::date,'2020-12-31'::date ,'1 month') as series1
       from plan1
	   left join customers c using (customer_id)
       group by 1),
       
 -- This code generate date for customers with plan journey 1,2       
p2 as (select customer_id, series1 
       from( select *, 
       row_number() over(partition by customer_id order by series1 ) as rankin
       from(select customer_id 
             ,GENERATE_SERIES(MIN(c.start_date)::date,
                              Max(c.start_date)::date ,
                              '1 month') as series1

             from plan1or2
             left join customers c using (customer_id)
             group by 1 )x1 )x2 
             where rankin = 1
             union 
             select customer_id, series1 from( select *, 
             row_number() over(partition by customer_id order by series1 desc ) as rankin
             from(select customer_id 
                   ,GENERATE_SERIES(MIN(c.start_date)::date,
                                    Max(c.start_date)::date ,
                                    '1 month') as series1

             from plan1or2
             left join customers c using (customer_id)
             group by 1 )x1 )x2 
             --where rankin <> 1
       
       union 
      select customer_id 
             ,GENERATE_SERIES(Max(c.start_date)::date,'2020-12-31'::date ,'1 month') as series1
             from plan1or2
             left join customers c using (customer_id)
             group by 1 
             order by 1,2),
             
 -- This code generate date for  customers with plan journey 1,2,3                         
p3 as (select customer_id, series1
       from( 
         
         select *, 
       row_number() over(partition by customer_id order by series1 ) as rankin
       from(select customer_id 
             ,GENERATE_SERIES(MIN(c.start_date)::date,
                              Max(c.start_date)::date ,
                              '1 month') as series1

             from plan123
             left join customers c using (customer_id)
             group by 1 )x1 )x2 
              where rankin in (1,2)
             union 
             select customer_id, series1 from( select *, 
             row_number() over(partition by customer_id order by series1 desc ) as rankin
             from(select customer_id 
                   ,GENERATE_SERIES(MIN(c.start_date)::date,
                                    Max(c.start_date)::date ,
                                    '1 month') as series1

             from plan123
             left join customers c using (customer_id)
             group by 1 )x1 )x2 
             -- where rankin <> 1
              
       
       union 
      select customer_id 
             ,Max(c.start_date) as series1
             from plan123
             left join customers c using (customer_id)
             group by 1 
             order by 1,2),
 
 -- This code remove plan 4 from all customers in plan1234
 p40 as (select customer_id, max(start_date) as mindate , b.maxdate
       from customers 
       
      left join (select customer_id, max(start_date) as maxdate 
                 from customers
                group by 1)b using (customer_id)
       where plan_id <>4
       group by 1,3),
       
 -- This code generate date for customer in plan1234           
 p4 as (select customer_id 
       ,GENERATE_SERIES(MIN(c.start_date)::date,
                        Max(c.start_date)::date ,
                        '1 month') as series1
           from plan1234
         left join customers c using (customer_id)
                   where c.plan_id <> 4
                   group by 1
           union 
          select customer_id 
               ,GENERATE_SERIES(mindate::date,
                                maxdate::date ,
                                '1 month') as series1
               from p40
               group by 1,2
               order by 1,2),
               
 -- This code create a prevplanid and price using the lag function
 df as  
  (select *,
     lag (plan_id) over(partition by customer_id order by start_date) as prevplanid
     ,lag (price) over(partition by customer_id order by start_date) as prevprice
     ,extract(year from start_date)::text || extract(month from start_date)::text as yearmonth
 from cust2),
 
 -- This code appends all tables to one 
 df1 as ( select tb1.customer_id,tb1.series1 as paymentdate  
			,tb1.customer_id:: text || tb1.series1:: text as pkey,
		  lag (tb1.series1) over(partition by tb1.customer_id order by tb1.series1) as prevdate
 
 from (      
       select * from p1
       union
       select * from p2
       union 
       select * from p3
       union 
       select * from p4)tb1

       order by 1,2),
 -- This code merge the plans table to our created table
  newdf as (select df1.customer_id as id, df1.paymentdate, df1.pkey, df.plan_id, df.start_date
          ,df.plan_name, df.price,df.prevprice,df1.prevdate,df.prevplanid,df.yearmonth,
          extract(year from df1.prevdate)::text || extract(month from df1.prevdate)::text as prevyearmonth
  from df1
 left join df on df1.customer_id = df.customer_id and df1.paymentdate = df.start_date
 order by 1,2),
 
-- This code fill down missing column using the last non null value 
 f1 as (select *, case when price is null then 
           (select price 
            from 
            newdf as i 
            where i.id = newdf.id and i.paymentdate < newdf.paymentdate
            and i.price is not null 
            order by i.paymentdate desc limit 1) else price end Pric
            
           , case when plan_name is null then 
           (select plan_name 
            from 
            newdf as i 
            where i.id = newdf.id and i.paymentdate < newdf.paymentdate
            and i.price is not null 
            order by i.paymentdate desc limit 1) else plan_name end Planname
            
             , case when plan_id is null then 
           (select plan_id 
            from 
            newdf as i 
            where i.id = newdf.id and i.paymentdate < newdf.paymentdate
            and i.price is not null 
            order by i.paymentdate desc limit 1) else plan_id end Planid
 from newdf),
 
 -- This code create a new price based on danny's request
 f2 as (select id,paymentdate,planname,planid
 ,case when planid = 2 and prevplanid = 1 and yearmonth = prevyearmonth then price - prevprice
       when plan_id = 3 and prevplanid = 1 and yearmonth = prevyearmonth then price - prevprice 
       else price end price2 
 from f1),
 
-- This code create a fill down for the new price column  
 f3 as (select id as customer_id,planid,planname,paymentdate

 , case when price2 is null then 
           (select price2 
            from 
            f2 as i 
            where i.id = f2.id and i.paymentdate < f2.paymentdate
            and i.price2 is not null 
            order by i.paymentdate desc limit 1) else price2 end Amount
  ,row_number() over(partition by id order by paymentdate) as paymentorder
 from f2)
 
 select * from f3
 where customer_id  IN (1,2,15,13,18,19,16,21,33,40,910,911,1000,996,206,219 )
 ///
