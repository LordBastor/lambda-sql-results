
# Infinite Lambda SQL Task

## Data quality investigation #1
- Let's start by grabbing the events which have user_ids which are not in our users table (you guys should **really** use fkey contraints here ;) )
```sql
SELECT event_id, 
       user_id 
FROM   user_interaction ui 
WHERE  user_id NOT IN (SELECT user_id 
                       FROM   users); 
```

The results we get are as follows:
|event_id                            |user_id|
|------------------------------------|-------|
|260C93F9-1BE3-4B35-99EA-9D78BBE3B63D|80     |
|FB523818-7A13-5C8E-A6AF-67AC34B21240|80     |
|4D94DFA3-E839-4EFE-A104-A745558BD570|80     |


- Lets see how many of each event type are either missattributed to the wrong user (or their user is missing)
```sql
SELECT event_type, 
       Count(event_type) AS event_count 
FROM   user_interaction ui 
WHERE  user_id NOT IN (SELECT user_id 
                       FROM   users) 
GROUP  BY event_type 
ORDER  BY event_count DESC; 
```
Results from the query:
|event_type|event_count|
|----------|-----------|
|VIEW      |2          |
|CLICK     |1          |

## Data quality investigation #2
- Let us now attempt to identify the duplicate entries by just grouping them (since they look like they should be unique per `event_id`/`user_id`/`event_type`/`event_time`
```sql
SELECT event_id, 
       user_id, 
       event_type, 
       event_time, 
       Count(*) 
FROM   user_interaction ui 
GROUP  BY event_id, 
          user_id, 
          event_type, 
          event_time 
HAVING Count(*) > 1 
```
The results indicate that we have had a duplicated `VIEW` and `EXIT` events
|event_id|user_id|event_type|event_time|count|
|--------|-------|----------|----------|-----|
|5F9CF4D8-6D12-331C-1224-109D188E9918|45|EXIT|2019-05-05 14:05:24|2|
|F36AABE3-0306-6669-36F0-5E6C255553FD|45|VIEW|2019-05-05 14:05:23|2|

The duplications occurred on the 5th of May 2019

## Analytics #1
- Let's see where our views are coming from
```sql
SELECT users.country, 
       Count(*) 
FROM   user_interaction ui 
       JOIN users 
       ON users.user_id = ui.user_id 
WHERE  event_type = 'VIEW' 
GROUP  BY users.country; 
```

Seems like the majority of our views have come from Deutschland
|country|count|
|-------|-----|
|DE|17|
|UK|3|

## Analytics #2
```sql
SELECT age, 
       ui.event_type, 
       Count(ui.event_type) AS event_count 
FROM   users 
       JOIN user_interaction ui 
       ON ui.user_id = users.user_id 
WHERE  event_type IN ( 'CLICK', 'VIEW' ) 
GROUP  BY age, 
          ui.event_type 
HAVING Count(ui.event_type) > 5 
       AND ui.event_type = 'VIEW' 
       OR Count(ui.event_type) > 10 
       AND ui.event_type = 'CLICK';
```

Results:
|age|event_type|event_count|
|---|----------|-----------|
|46-80|CLICK|35|
|18-45|CLICK|19|
|18-45|VIEW|6|
|46-80|VIEW|14|



##  Analytics #3

```sql
CREATE TABLE account_invoice_records AS 
SELECT ih.invoice_number , 
       ih.account_number , 
       ih.invoice_created_date , 
       ih.invoice_due_date , 
       ii.product_number , 
       ii.net_amount , 
       ii.tax_amount 
FROM   invoice_headers ih -- 10m+ records 
JOIN   invoice_items ii   -- 100m+ records where 1=2 
and    ih.account_number = 1234;
```

The above query would prove problematic.
It will probably cause metadata lock issues, prevent other queries from executing until it's finished or it might even be outright rejected depending on the DB engine setup, not to mention preventing other queries to read from the destination table.


I would honestly go for a `SELECT ... INTO` statement, get the data in some sort of dumpfile and write it back using `LOAD DATA`

This would ensure we avoid locking a bunch of tables while doing a crazy operation.