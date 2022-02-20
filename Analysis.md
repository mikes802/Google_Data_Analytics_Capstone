## Analysis
Having uploaded all of the necessary tables to BigQuery and changed all of the relevant 'dates' columns to a data type that BigQuery recognized (`TIMESTAMP`), I proceeded to analyze the data. Note that the database name may change in some of the queries. This is because I reorganized the database after successfully converting the data types. Some queries were accomplished before that change and others were carried out afterwards.

I will first run a few queries to gain an understanding of the number of participants for each table.

In the weight table, there are only 8 distinct IDs.
```
SELECT 
    COUNT(DISTINCT Id)
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.weight_log_info_clean`;
```
```
SELECT 
    DISTINCT Id
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.weight_log_info_clean`;
```
In the sleep table, there are 24 distinct IDs.
```
SELECT 
    COUNT(DISTINCT Id)
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.sleep_day_cleaned`;
```
```
SELECT 
    DISTINCT Id
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.sleep_day_cleaned`;
```
Curiously, in the daily activity table, there are 33 distinct IDs. We were told in the description of the data on Kaggle that there are only 30 participants.
```
SELECT 
    COUNT(DISTINCT Id)
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.daily_activity`;
```
```
SELECT 
    DISTINCT Id
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.daily_activity`
ORDER BY Id DESC;
```
After reviewing a few more tables this way, I discover that many of them show 33 distinct IDs.

I want to look a little deeper at the `daily_activity` table and decide to run a query to find the `MIN`, `MAX`, and the range of total steps in the `TotalSteps` column.
```
WITH max_min_values AS (
  SELECT 
    MIN(TotalSteps) AS min_steps,
    MAX(TotalSteps) AS max_steps,
  FROM 
    `tribal-isotope-321016.fitbit_tracker_data.daily_activity`
)
SELECT 
  min_steps,
  max_steps,
  max_steps - min_steps AS range_steps
FROM max_min_values;
```
Of course I discover that the minimum number of steps is 0. I want to find out what the minumum number of steps are when it is not 0.
```
WITH max_min_values AS (
    SELECT 
        MIN(TotalSteps) AS min_steps,
        MAX(TotalSteps) AS max_steps,
    FROM (
        SELECT *
        FROM  
            `tribal-isotope-321016.fitbit.daily_activity`
        WHERE 
            TotalSteps != 0
    )
)
SELECT 
    min_steps,
    max_steps,
    max_steps - min_steps AS range_steps
FROM max_min_values;
```
The minimum number of steps is now 4, which is still very low. The maximum number is 36,019, giving a range of 36,015.

I want to look at the stats for the weight table. I discover that I cannot make a calculation for median in BigQuery if I'm looking for other stat numbers at the same time. I do this in a separate query and use `LIMIT 1` so as not to get a long list of the same number.
```
SELECT 
    MIN(WeightPounds) AS minimum_weight_lb,
    MAX(WeightPounds) AS maximum_weight_lb,
    AVG(WeightPounds) AS mean_weight_lb,
    APPROX_TOP_COUNT(WeightPounds, 1) AS mode_value,
    STDDEV(WeightPounds) AS standard_deviation,
    VARIANCE(WeightPounds) AS variance_value 
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.weight_log_info_clean`;
```
```
SELECT 
    PERCENTILE_CONT(WeightPounds, 0.5) OVER () AS median_value
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.weight_log_info_clean`
LIMIT 1;
```
The following information is returned. Note that I did not round in the query. These values that are returned seem to be appropriate.

| Variable | Value |
| --- | --- |
| minimum_weight_lb | 115.96 |
| maximum_weight_lb | 294.32 |
| mean_weight_lb | 158.81 |
| mode_value | 188.50 |
| mode_value.count | 5 |
| median_value | 137.79 |
|standard_deviation | 30.70 |
| variance_value | 942.21 |


