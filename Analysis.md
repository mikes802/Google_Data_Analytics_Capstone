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
Curiously, in the `daily_activity` table, there are 33 distinct IDs. We are told in the description of the data on Kaggle that there are only 30 participants.
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
The following information is returned. Note that I did not round in the query. The values that are returned seem to be appropriate.

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

I now want to see if I can find any trends between the data from the activity and sleep tables, `daily_activity` and `sleep_day_cleaned` respectively. First, I will confirm that there are no duplicates in the `daily_activity` table.
```
SELECT  
    COUNT(*)
FROM `tribal-isotope-321016.fitbit_tracker_data.daily_activity`;
```
```
SELECT 
    COUNT(*)
FROM (
    SELECT 
        DISTINCT *
    FROM `tribal-isotope-321016.fitbit_tracker_data.daily_activity`
) AS subquery;
```
I want to join a table populated with the average values for each participant from the `daily_activity` table with a table populated with the average values for each participant from the `sleep_day_cleaned` table. I know there are 3 duplicates in the `sleep_day_cleaned` table, so I will need to take that into consideration when constructing this query. I will create these two tables, `avg_daily_data` and `avg_sleep_data`.
```
DROP TABLE IF EXISTS `tribal-isotope-321016.fitbit_tracker_data.avg_daily_data`;

CREATE TABLE `tribal-isotope-321016.fitbit_tracker_data.avg_daily_data` AS (
    SELECT
        Id,
        COUNT(ActivityDate) AS number_entries,
        COUNT(DISTINCT ActivityDate) AS number_dates,
        ROUND(AVG(TotalSteps), 2) AS avg_tot_steps,
        ROUND(AVG(TotalDistance), 2) AS avg_tot_distance,
        ROUND(AVG(VeryActiveDistance), 2) AS avg_very_act_dist,
        ROUND(AVG(ModeratelyActiveDistance), 2) AS avg_mod_act_dist,
        ROUND(AVG(LightActiveDistance), 2) AS avg_lit_act_dist,
        ROUND(AVG(VeryActiveMinutes), 2) AS avg_very_act_min,
        ROUND(AVG(FairlyActiveMinutes), 2) AS avg_fair_act_min,
        ROUND(AVG(LightlyActiveMinutes), 2) AS avg_lit_act_min,
        ROUND(AVG(SedentaryMinutes), 2) AS avg_sed_min,
        ROUND(AVG(Calories), 2) AS avg_calories
    FROM 
    `tribal-isotope-321016.fitbit_tracker_data.daily_activity`
    GROUP BY 
        Id
);
```
```
DROP TABLE IF EXISTS `tribal-isotope-321016.fitbit_tracker_data.avg_sleep_data`;

CREATE TABLE `tribal-isotope-321016.fitbit_tracker_data.avg_sleep_data` AS (
    SELECT 
        Id,
        COUNT(SleepDay) AS number_sleep_days,
        ROUND(AVG(TotalMinutesAsleep), 2) AS avg_min_sleep,
        ROUND(AVG(TotalTimeInBed), 2) AS avg_time_in_bed
    FROM (
        SELECT 
            DISTINCT *
        FROM `tribal-isotope-321016.fitbit_tracker_data.sleep_day`
    )
    GROUP BY 
        Id
);
```
Now I will join them using an `INNER JOIN`. This will drop the IDs of the participants who did not have any `SleepDay` records. Every participant who shows up in this table will have both daily activity and sleep data.
```
SELECT 
    daily_activity_data.*, sleep_day_data.*
FROM 
    `tribal-isotope-321016.fitbit_tracker_data.avg_daily_data` AS daily_activity_data
INNER JOIN
    `tribal-isotope-321016.fitbit_tracker_data.avg_sleep_data` AS sleep_day_data ON
    daily_activity_data.Id = sleep_day_data.Id;
```
24 rows are returned. I do find a few positive relationships between variables after exploring the data on Google's Data Studio.

### Number of Sleep Days vs Average Light Active Distance

![Sleep Days v Ave Lit Act Dist](https://user-images.githubusercontent.com/99853599/154879905-cbccfb3b-795f-4205-860e-4d0d9a0255fe.PNG)

### Number of Sleep Days vs Average Light Active Minutes

![Sleep Days v Ave Lit Act Min](https://user-images.githubusercontent.com/99853599/154880048-03adcf73-75b6-4bb2-a9fa-dbcb16bc0b61.PNG)

### Number of Sleep Days vs Average Moderately Active Distance

![Sleep Days v Ave Mod Act Dist](https://user-images.githubusercontent.com/99853599/154881242-625a6ac1-ad45-4fe3-9efd-f77c3df67d90.PNG)

### Number of Sleep Days vs Average Fairly Active Minutes

![Sleep Days v Fair Act Min](https://user-images.githubusercontent.com/99853599/154880883-7eec1508-7fad-41a4-b6e6-823a4b8c69dd.PNG)

### Number of Sleep Days vs Average Total Distance

![Sleep Days v Ave Total Dist](https://user-images.githubusercontent.com/99853599/154880524-e82e0548-d9ae-4227-b8f3-ac9cabd2d8af.PNG)

There is also one clear negative relationship.

### Number of Sleep Days vs Average Sedentary Minutes

![Sleep Days v Ave Sed Min](https://user-images.githubusercontent.com/99853599/154880221-6d9ec679-3d94-4a73-a96b-791a6c7dedf1.PNG)

I notice that there is a fairly clear demarcation between those who frequently logged sleep days and those who barely did or simply did not. I want to check the average numbers between the two groups. I will demarcate the groups at <= 5 days and >= 8 days.

Group A is >= 8 days and 16 participants meet this criteria.
```
WITH cte_table AS (
    SELECT 
        daily_activity_data.*, sleep_day_data.number_sleep_days, sleep_day_data.avg_min_sleep, sleep_day_data.avg_time_in_bed
    FROM 
        `tribal-isotope-321016.fitbit.avg_daily_data` AS daily_activity_data
    INNER JOIN
        `tribal-isotope-321016.fitbit.avg_sleep_data` AS sleep_day_data ON
        daily_activity_data.Id = sleep_day_data.Id
)
SELECT 
    COUNT(DISTINCT Id),
    ROUND(AVG(number_dates), 2) AS gave_num_dates,
    ROUND(AVG(avg_tot_steps), 2) AS gavg_tot_steps,
    ROUND(AVG(avg_tot_distance), 2) AS gavg_tot_dist,
    ROUND(AVG(avg_very_act_dist), 2) AS gavg_very_act_dist,
    ROUND(AVG(avg_mod_act_dist), 2) AS gavg_mod_act_dist,
    ROUND(AVG(avg_lit_act_dist), 2) AS gavg_lit_act_dist,
    ROUND(AVG(avg_very_act_min), 2) AS gavg_very_act_min,
    ROUND(AVG(avg_fair_act_min), 2) AS gavg_fair_act_min,
    ROUND(AVG(avg_lit_act_min), 2) AS gavg_lit_act_min,
    ROUND(AVG(avg_sed_min), 2) AS gavg_sed_min,
    ROUND(AVG(avg_calories), 2) AS gavg_calories
FROM cte_table
WHERE number_sleep_days >= 8;
```

| Group A Variable | Group A Value |
| --- | --- |
| number of participants | 16 |
| gave_num_dates | 29.75 |
| gavg_tot_steps | 7850.51 |
| gavg_tot_dist | 5.53 |
| gavg_very_act_dist | 1.27 |
| gavg_mod_act_dist | 0.7 |
| gavg_lit_act_dist | 3.49 |
| gavg_very_act_min | 22.02 |
| gavg_fair_act_min | 16.88 |
| gavg_lit_act_min	| 197.84 |
| gavg_sed_min | 802.92 |
| gavg_calories | 2314.19 |	

Group B is <= 5 days and 17 participants meet this criteria. For this query I will have to do a `FULL OUTER JOIN` to bring back in those participants whose sleep data `IS NULL`. In other words, they didn't log any sleep days at all.
```
WITH cte_table AS (
    SELECT 
        daily_activity_data.*, sleep_day_data.number_sleep_days, sleep_day_data.avg_min_sleep, sleep_day_data.avg_time_in_bed
    FROM 
        `tribal-isotope-321016.fitbit.avg_daily_data` AS daily_activity_data
    FULL OUTER JOIN
        `tribal-isotope-321016.fitbit.avg_sleep_data` AS sleep_day_data ON
        daily_activity_data.Id = sleep_day_data.Id
)
SELECT 
    COUNT(DISTINCT Id),
    ROUND(AVG(number_dates), 2) AS gave_num_dates,
    ROUND(AVG(avg_tot_steps), 2) AS gavg_tot_steps,
    ROUND(AVG(avg_tot_distance), 2) AS gavg_tot_dist,
    ROUND(AVG(avg_very_act_dist), 2) AS gavg_very_act_dist,
    ROUND(AVG(avg_mod_act_dist), 2) AS gavg_mod_act_dist,
    ROUND(AVG(avg_lit_act_dist), 2) AS gavg_lit_act_dist,
    ROUND(AVG(avg_very_act_min), 2) AS gavg_very_act_min,
    ROUND(AVG(avg_fair_act_min), 2) AS gavg_fair_act_min,
    ROUND(AVG(avg_lit_act_min), 2) AS gavg_lit_act_min,
    ROUND(AVG(avg_sed_min), 2) AS gavg_sed_min,
    ROUND(AVG(avg_calories), 2) AS gavg_calories
FROM cte_table
WHERE number_sleep_days IS NULL OR
    number_sleep_days <= 5;
```

| Group B Variable | Group B Value |
| --- | --- |
| number of participants | 17 |
| gave_num_dates | 27.29 |
| gavg_tot_steps | 7207.52 |
| gavg_tot_dist | 5.27 |
| gavg_very_act_dist | 1.62 |
| gavg_mod_act_dist | 0.42 |
| gavg_lit_act_dist | 3.15 |
| gavg_very_act_min | 18.7 |
| gavg_fair_act_min | 9.85 |
| gavg_lit_act_min	| 185.57 |
| gavg_sed_min | 1183.83 |
| gavg_calories | 2252.57 |	

Looking at how Group B compares to Group A, there are some notable results:
1. The two groups are nearly split down the middle, with 16 participants logging 8 or more sleep days and 17 participants logging 5 days or less.
2. Group B averages 47% more sedentary minutes than Group A.
3. Group B averages 42% fewer fairly active minutes than Group A.
4. Group B averages 40% less in moderately active distance compared to Group A.
5. In fact, Group B averages less than Group A in every category except for sedentary minutes, as mentioned above, and very active distance, which is 28% more than Group A.

We can see how Group B compares to Group A here in this chart:

![Group B Comparison](https://user-images.githubusercontent.com/99853599/155254929-eee5241e-7ff9-47d4-a706-fbdc008ae5db.PNG)

I'm going to approach this from another direction using SQL code provided by Google as follows:

>  -- Say we want to do an analysis based upon daily data, this could help us to find tables that might be at the day level
> ``` 
>  SELECT
>  DISTINCT table_name
>  FROM
>  `tribal-isotope-321016.fitbit.INFORMATION_SCHEMA.COLUMNS`
>  WHERE
>  REGEXP_CONTAINS(LOWER(table_name),"day|daily");
>  ```

| Row |	table_name	|
| --- | --- |
|1	| daily_calories |
| 2 |	daily_activity |
|3 | daily_intensities |
| 4 |	 daily_steps |
| 5 |	 sleep_day |

>  -- Now that we have a list of tables we should look at the columns that are shared among the tables
>  ```
> SELECT
>   column_name,
>   data_type,
> COUNT(table_name) AS table_count
> FROM
>   `tribal-isotope-321016.fitbit.INFORMATION_SCHEMA.COLUMNS`
> WHERE
>   REGEXP_CONTAINS(LOWER(table_name),"day|daily")
> GROUP BY
>   1, 2;

| Row | column_name              | data_type | table_count |
|-----|--------------------------|-----------|-------------|
| 1   | Id                       | INT64     |           5 |
| 2   | ActivityDay              | DATE      |           3 |
| 3   | Calories                 | INT64     |           2 |
| 4   | ActivityDate             | DATE      |           1 |
| 5   | TotalSteps               | INT64     |           1 |
| 6   | TotalDistance            | FLOAT64   |           1 |
| 7   | TrackerDistance          | FLOAT64   |           1 |
| 8   | LoggedActivitiesDistance | FLOAT64   |           1 |
| 9   | VeryActiveDistance       | FLOAT64   |           2 |
| 10  | ModeratelyActiveDistance | FLOAT64   |           2 |
| 11  | LightActiveDistance      | FLOAT64   |           2 |
| 12  | SedentaryActiveDistance  | FLOAT64   |           2 |
| 13  | VeryActiveMinutes        | INT64     |           2 |
| 14  | FairlyActiveMinutes      | INT64     |           2 |
| 15  | LightlyActiveMinutes     | INT64     |           2 |
| 16  | SedentaryMinutes         | INT64     |           2 |
| 17  | StepTotal                | INT64     |           1 |
| 18  | SleepDay                 | DATE      |           1 |
| 19  | TotalSleepRecords        | INT64     |           1 |
| 20  | TotalMinutesAsleep       | INT64     |           1 |
| 21  | TotalTimeInBed           | INT64     |           1 |

The next SQL query provided by Google shows specifically which tables share which columns. This information will be used to join the five tables above.

>```
> SELECT
>  column_name,
>  table_name,
>  data_type
> FROM
>  `tribal-isotope-321016.fitbit.INFORMATION_SCHEMA.COLUMNS`
> WHERE
>  REGEXP_CONTAINS(LOWER(table_name),"day|daily")
>  AND column_name IN (
>  SELECT
>   column_name
>  FROM
>   `tribal-isotope-321016.fitbit.INFORMATION_SCHEMA.COLUMNS`
>  WHERE
>   REGEXP_CONTAINS(LOWER(table_name),"day|daily")
>  GROUP BY
>   1
>  HAVING
>   COUNT(table_name) >=2)
> ORDER BY
>  1;
> ```

| Row | column_name              | table_name        | data_type |
|-----|--------------------------|-------------------|-----------|
| 1   | ActivityDay              | daily_calories    | DATE      |
| 2   | ActivityDay              | daily_intensities | DATE      |
| 3   | ActivityDay              | daily_steps       | DATE      |
| 4   | Calories                 | daily_calories    | INT64     |
| 5   | Calories                 | daily_activity    | INT64     |
| 6   | FairlyActiveMinutes      | daily_activity    | INT64     |
| 7   | FairlyActiveMinutes      | daily_intensities | INT64     |
| 8   | Id                       | daily_calories    | INT64     |
| 9   | Id                       | daily_activity    | INT64     |
| 10  | Id                       | daily_intensities | INT64     |
| 11  | Id                       | daily_steps       | INT64     |
| 12  | Id                       | sleep_day         | INT64     |
| 13  | LightActiveDistance      | daily_activity    | FLOAT64   |
| 14  | LightActiveDistance      | daily_intensities | FLOAT64   |
| 15  | LightlyActiveMinutes     | daily_activity    | INT64     |
| 16  | LightlyActiveMinutes     | daily_intensities | INT64     |
| 17  | ModeratelyActiveDistance | daily_activity    | FLOAT64   |
| 18  | ModeratelyActiveDistance | daily_intensities | FLOAT64   |
| 19  | SedentaryActiveDistance  | daily_activity    | FLOAT64   |
| 20  | SedentaryActiveDistance  | daily_intensities | FLOAT64   |
| 21  | SedentaryMinutes         | daily_activity    | INT64     |
| 22  | SedentaryMinutes         | daily_intensities | INT64     |
| 23  | VeryActiveDistance       | daily_activity    | FLOAT64   |
| 24  | VeryActiveDistance       | daily_intensities | FLOAT64   |
| 25  | VeryActiveMinutes        | daily_activity    | INT64     |
| 26  | VeryActiveMinutes        | daily_intensities | INT64     |

Here is the join query:

> ```
> SELECT
>  A.Id,
>  A.Calories,
>  * EXCEPT(Id,
>    Calories,
>    ActivityDay,
>    SleepDay,
>    SedentaryMinutes,
>    LightlyActiveMinutes,
>    FairlyActiveMinutes,
>    VeryActiveMinutes,
>    SedentaryActiveDistance,
>    LightActiveDistance,
>    ModeratelyActiveDistance,
>    VeryActiveDistance),
>  I.SedentaryMinutes,
>  I.LightlyActiveMinutes,
>  I.FairlyActiveMinutes,
>  I.VeryActiveMinutes,
>  I.SedentaryActiveDistance,
>  I.LightActiveDistance,
>  I.ModeratelyActiveDistance,
>  I.VeryActiveDistance
> FROM
>  `tribal-isotope-321016.fitbit.daily_activity` A
> LEFT JOIN
>  `tribal-isotope-321016.fitbit.daily_calories` C
> ON
>  A.Id = C.Id
>  AND A.ActivityDate=C.ActivityDay
>  AND A.Calories = C.Calories
> LEFT JOIN
>  `tribal-isotope-321016.fitbit.daily_intensities` I
> ON
>  A.Id = I.Id
>  AND A.ActivityDate=I.ActivityDay
>  AND A.FairlyActiveMinutes = I.FairlyActiveMinutes
>  AND A.LightActiveDistance = I.LightActiveDistance
>  AND A.LightlyActiveMinutes = I.LightlyActiveMinutes
>  AND A.ModeratelyActiveDistance = I.ModeratelyActiveDistance
>  AND A.SedentaryActiveDistance = I.SedentaryActiveDistance
>  AND A.SedentaryMinutes = I.SedentaryMinutes
>  AND A.VeryActiveDistance = I.VeryActiveDistance
>  AND A.VeryActiveMinutes = I.VeryActiveMinutes
> LEFT JOIN
>  `tribal-isotope-321016.fitbit.daily_steps` S
> ON
>  A.Id = S.Id
>  AND A.ActivityDate=S.ActivityDay
> LEFT JOIN
>  `tribal-isotope-321016.fitbit.sleep_day` Sl
> ON
>  A.Id = Sl.Id
>  AND A.ActivityDate=Sl.SleepDay;
>  ```

I will save this table as `dates_joined` and then use the following query to see how many records actually include information on sleep.
```
SELECT *
FROM `tribal-isotope-321016.fitbit.dates_joined`
WHERE TotalSleepRecords IS NOT NULL;
```
413 records are returned. 

From `dates_joined`, I will check the number of days records were taken and the total number of sleep records for each ID.
```
SELECT Id,
    COUNT(Id) AS Number_Days,
    SUM(TotalSleepRecords) Total_Sleep_Records
FROM `tribal-isotope-321016.fitbit.dates_joined`
WHERE TotalSleepRecords IS NOT NULL
GROUP BY 1
ORDER BY 3 DESC;
```
24 out of 33 participants had at least one sleep record. This matches the results I received when I completed an `INNER JOIN` on the `avg_daily_data` and `avg_sleep_data` tables.

The following query should tell me, percent-wise, how many participants logged sleep records for a certain number of days.
```
WITH cte_table AS (
SELECT Id,
    COUNT(Id) AS Number_Days,
    SUM(TotalSleepRecords) Total_Sleep_Records
FROM `tribal-isotope-321016.fitbit.dates_joined`
WHERE TotalSleepRecords IS NOT NULL
GROUP BY 1
ORDER BY 3 DESC
)
SELECT 
    Number_Days,
    COUNT(*) as count_number,
    ROUND(
        100 * COUNT(*) / SUM(COUNT(*)) OVER(),
        2
    )   AS percentage,
    SUM(COUNT(*)) OVER() AS total_count
FROM cte_table
GROUP BY 1
ORDER BY 1;
```
| Row | Number_Days | count_number | percentage | total_count |
|-----|-------------|--------------|------------|-------------|
| 1   |           1 |            1 |       4.17 |          24 |
| 2   |           2 |            1 |       4.17 |          24 |
| 3   |           3 |            3 |       12.5 |          24 |
| 4   |           4 |            1 |       4.17 |          24 |
| 5   |           5 |            2 |       8.33 |          24 |
| 6   |           8 |            1 |       4.17 |          24 |
| 7   |          15 |            2 |       8.33 |          24 |
| 8   |          18 |            1 |       4.17 |          24 |
| 9   |          24 |            2 |       8.33 |          24 |
| 10  |          25 |            1 |       4.17 |          24 |
| 11  |          26 |            2 |       8.33 |          24 |
| 12  |          28 |            4 |      16.67 |          24 |
| 13  |          31 |            2 |       8.33 |          24 |
| 14  |          32 |            1 |       4.17 |          24 |

One of the results from this query shows 32 days. I want to double-check the dates.
```
SELECT 
    MIN(ActivityDate) earliest_date,
    MAX(ActivityDate) latest_date,
    DATE_DIFF(MAX(ActivityDate), MIN(ActivityDate), DAY) + 1 AS total_days 
FROM `tribal-isotope-321016.fitbit.daily_activity`;
```
| earliest_date | latest_date | total_days |
| --- | --- | --- |
| 2016-04-12 |	2016-05-12 |	31 |

Of course this begs the question: how did someone have 32 days? I know duplicates exist, so I need to check for that. I will check the specific ID.
```
SELECT * 
FROM `tribal-isotope-321016.fitbit.dates_joined` 
WHERE Id = 8378563200 
ORDER BY ActivityDate;
```
It looks like there is a duplicate row on 4/25. Here's a truncated table:

| Row | Id         | Calories | ActivityDate | TotalSteps | TotalDistance    | TrackerDistance  | LoggedActivitiesDistance | StepTotal |
|-----|------------|----------|--------------|------------|------------------|------------------|--------------------------|-----------|
| 14  | 8378563200 |     4005 | 2016-04-25   |      12405 | 9.84000015258789 | 9.84000015258789 |          2.0921471118927 |     12405 |
| 15  | 8378563200 |     4005 | 2016-04-25   |      12405 | 9.84000015258789 | 9.84000015258789 |          2.0921471118927 |     12405 |

I will use the following query to find duplicates in `dates_joined`.
```
SELECT 
    Id,
    Calories,
    ActivityDate,
    TotalSteps,
    TotalDistance,
    TrackerDistance,
    LoggedActivitiesDistance,
    StepTotal,
    TotalSleepRecords,
    TotalMinutesAsleep,
    TotalTimeInBed,
    SedentaryMinutes,
    LightlyActiveMinutes,
    FairlyActiveMinutes,
    VeryActiveMinutes,
    SedentaryActiveDistance,
    LightActiveDistance,
    ModeratelyActiveDistance,
    VeryActiveDistance,
    COUNT(*) AS frequency
FROM
    `tribal-isotope-321016.fitbit.dates_joined`
GROUP BY
    Id,
    Calories,
    ActivityDate,
    TotalSteps,
    TotalDistance,
    TrackerDistance,
    LoggedActivitiesDistance,
    StepTotal,
    TotalSleepRecords,
    TotalMinutesAsleep,
    TotalTimeInBed,
    SedentaryMinutes,
    LightlyActiveMinutes,
    FairlyActiveMinutes,
    VeryActiveMinutes,
    SedentaryActiveDistance,
    LightActiveDistance,
    ModeratelyActiveDistance,
    VeryActiveDistance
ORDER BY frequency DESC;
```
There are three duplicates, just like I had found previously. Using the following query, I will clean up the duplicate records.
```
WITH cte_table AS (
SELECT
    Id,
    COUNT(Id) AS Number_Days,
    SUM(TotalSleepRecords) Total_Sleep_Records
FROM (
    SELECT 
        Id,
        TotalSleepRecords,
    FROM
        `tribal-isotope-321016.fitbit.dates_joined`
    GROUP BY
        Id,
        Calories,
        ActivityDate,
        TotalSteps,
        TotalDistance,
        TrackerDistance,
        LoggedActivitiesDistance,
        StepTotal,
        TotalSleepRecords,
        TotalMinutesAsleep,
        TotalTimeInBed,
        SedentaryMinutes,
        LightlyActiveMinutes,
        FairlyActiveMinutes,
        VeryActiveMinutes,
        SedentaryActiveDistance,
        LightActiveDistance,
        ModeratelyActiveDistance,
        VeryActiveDistance
    )
WHERE TotalSleepRecords IS NOT NULL
GROUP BY 1
ORDER BY 2 DESC
)
SELECT 
    Number_Days,
    COUNT(*) as count_number,
    ROUND(
        100 * COUNT(*) / SUM(COUNT(*)) OVER(),
        2
    )   AS percentage,
    SUM(COUNT(*)) OVER() AS total_count
FROM cte_table
GROUP BY 1
ORDER BY 1;
```
Now the most number of days is 31. This is the table that generates:

| Row | Number_Days | count_number | percentage | total_count |
|-----|-------------|--------------|------------|-------------|
| 1   |           1 |            1 |       4.17 |          24 |
| 2   |           2 |            1 |       4.17 |          24 |
| 3   |           3 |            3 |       12.5 |          24 |
| 4   |           4 |            1 |       4.17 |          24 |
| 5   |           5 |            2 |       8.33 |          24 |
| 6   |           8 |            1 |       4.17 |          24 |
| 7   |          15 |            2 |       8.33 |          24 |
| 8   |          18 |            1 |       4.17 |          24 |
| 9   |          23 |            1 |       4.17 |          24 |
| 10  |          24 |            1 |       4.17 |          24 |
| 11  |          25 |            1 |       4.17 |          24 |
| 12  |          26 |            2 |       8.33 |          24 |
| 13  |          27 |            1 |       4.17 |          24 |
| 14  |          28 |            3 |       12.5 |          24 |
| 15  |          31 |            3 |       12.5 |          24 |

I'm seeing a similar table to what I had before when looking at my `avg_daily_data` and `avg_sleep_data` tables. Using some addition, I find from here that:
1. About 1/3 of those who use the sleep functionality use it from 1 - 5 days.
2. About 17% use the sleep functionality at least 8 days but not more than 18 days.
3. 50% use the sleep funtionality 23 days or more.

I will run the following query to see if I can find any interesting trends between number of logged sleep days and some other variable:
```
WITH cte_table AS (
SELECT *
FROM
    `tribal-isotope-321016.fitbit.dates_joined`
GROUP BY
    Id,
    Calories,
    ActivityDate,
    TotalSteps,
    TotalDistance,
    TrackerDistance,
    LoggedActivitiesDistance,
    StepTotal,
    TotalSleepRecords,
    TotalMinutesAsleep,
    TotalTimeInBed,
    SedentaryMinutes,
    LightlyActiveMinutes,
    FairlyActiveMinutes,
    VeryActiveMinutes,
    SedentaryActiveDistance,
    LightActiveDistance,
    ModeratelyActiveDistance,
    VeryActiveDistance
)
SELECT
    Id,
    COUNT(Id) AS number_days_active,
    SUM (CASE 
            WHEN TotalSleepRecords IS NOT NULL THEN 1
        ELSE 
        0
    END
    ) AS number_sleep_days,
    MIN(ActivityDate) first_day,
    MAX(ActivityDate) last_day,
    ROUND(AVG(TotalSteps),2) avg_tot_steps,
    ROUND(AVG(SedentaryMinutes),2) avg_sed_min,
    ROUND(AVG(LightlyActiveMinutes),2) avg_light_min,
    ROUND(AVG(FairlyActiveMinutes),2) avg_fairly_act_min,
    ROUND(AVG(VeryActiveMinutes),2) avg_very_act_min,
    ROUND(AVG(SedentaryActiveDistance),2) avg_sed_dist,
    ROUND(AVG(LightActiveDistance),2) avg_light_dist,
    ROUND(AVG(ModeratelyActiveDistance),2) avg_mod_act_dist,
    ROUND(AVG(VeryActiveDistance),2) avg_very_act_dist
FROM cte_table
GROUP BY 1
ORDER BY 1 DESC;
```
After exploring the data in Google Data Studio, I find that the negative relationship between `number_sleep_days` and `avg_sed_min` is the most striking, as it was above. Here is the graph:

### Number of Sleep Days vs Average Sedentary Minutes (Includes All Participants)

![Number Sleep Days v Ave Sed Min](https://user-images.githubusercontent.com/99853599/155448434-c896e395-a3e9-4ebd-a1cd-5e7ed1ee235f.PNG)

This is a steeper trend line than the previous graph because the participants who don't have any `number_sleep_day` values all have fairly high values for `avg_sed_min`. The lowest value is 1077.55. In fact, just by looking at the graph itself, aside from one outlier who logged 15 sleep days but has an `avg_sed_min` value of 1060.48, all of the participants who logged at least 15 sleep days fall between `avg_sed_min` values of 662.32 and 850.45. The remaining participants, who logged 8 sleep days or less, have `avg_sed_min` values of 1055.35 and higher.

I will use the insights I gained above to answer the questions in the [post-analysis](/Post-Analysis.md#post-analysis) and to draft my presentation.

Back to [table of contents](/README.md#table-of-contents).
