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

I now want to see if I can find any trends between the data from activity and sleep tables, `daily_activity` and `sleep_day_cleaned` respectively. First, I will confirm that there are no duplicates in the `daily_activity` table.
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
FULL OUTER JOIN
    `tribal-isotope-321016.fitbit_tracker_data.avg_sleep_data` AS sleep_day_data ON
    daily_activity_data.Id = sleep_day_data.Id;
```
I do find a few positive relationships between variables after exploring the data on Google's Data Studio.

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

I notice that there is a fairly clear demarcation between those who actively used the sleep functions and those who barely used them or didn't use them at all. I want to check the average numbers between the two groups. I will demarcate the groups at <= 5 days and >= 8 days.

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

Group B is <= 5 days and 17 participants meet this criteria. For this query I will have to do a `FULL OUTER JOIN` to bring back in those participants whose sleep data `IS NULL`. In other words, they didn't use the sleep functionality at all.
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
