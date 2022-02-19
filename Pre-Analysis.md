# Pre-Analysis
## Ask 
### Guiding Questions
*What is the problem you are trying to solve?* 

Gain insight into consumer use trends of smart devices.

*How can your insights drive business decisions?*

Use the insights gained to give high-level recommendations for Bellabeat’s marketing strategy. Insights can be trends that are discovered that could identify a specific target audience that is being under-utilized or a service that needs expanding.

### Key Tasks
*1. Identify the business task*

Identify insight into consumer use trends of smart devices that will inform Bellabeat’s marketing strategy for one Bellabeat product.

*2. Consider the key stakeholders*

The key stakeholders are:
1. Urska Srsen, Bellabeat’s cofounder and Chief Creative Officer
2. Sando Mur, Mathematician and Bellabeat’s cofounder; key member of the Bellabeat executive team
3. Bellabeat marketing analysis team
## Prepare
### Guiding Questions
*Where is your data stored?*

The [FitBit Fitness Tracker Data](https://www.kaggle.com/arashnic/fitbit) (CCO: Public Domain, dataset available through [Mobius](https://www.kaggle.com/arashnic)) is from Kaggle via [Zenodo](https://zenodo.org/record/53894#.Yg5oXN_MKUm). Zenodo, according to Wikipedia, is an open-access repository operated by CERN and developed under the European OpenAIRE program. I downloaded it from Kaggle in csv format.

*How is the data organized? Is it in long or wide format?*

The data is stored in long format, with one ID storing multiple instances of data.

*Are there issues with bias or credibility in this data? Does your data __ROCCC__?*

The data seems __reliable__, given its [source](https://www.kaggle.com/arashnic/fitbit). Since it is data direct from the users, it is __original__. I have validated the Kaggle set with the original source on Zenodo. The data is __comprehensive__ and should help us answer the question about trends in smart device usage. The data is not __current__, but it was the dataset we were requested to analyze. The dataset is __cited__: Furberg, R., Brinton, J., Keating, M., & Ortiz, A. (2016). The data does ROCCC except for not being current. I am concerned that the sample size is too small to make any meaningful insight. Thirty is the minimum sample size, and we have a sample size of exactly thirty. Also, it looks like the users who opted to provide their data to this project chose to do so, so it is not exactly a “random” sample set, but self-selected. There is nothing in the data that would tell me whether or not the population size is unbiased in any way. I don’t know age, race, weight, gender, etc.

*How are you addressing licensing, privacy, security, and accessibility?*

The collection of this data, according to the metadata, was consented to by the Fitbit users. The data seems to have been anonymized with no readily identifiable PII (personally identifiable information). I am storing the dataset on my personal computer which is only accessible by me and can only be accessed with a password.

*How does it help you answer your question?*

Our task is to find insight into how users use the smart devices. At first glance, we have a lot of data concerning how often the device is used or tracks activity during the day, and perhaps from this we can glean insight on usage. 

*Are there any problems with the data?*

There is no metadata, so there are certain values about which I am completely confused. The sample size is smaller than ideal. Also, the data is old, from 2016. It would be best to find Fitbit data that is newer, larger, and better explained. I looked online and cannot find such a dataset.
## Process
### Guiding Questions
*What tools are you choosing and why?*

I will use SQL in BigQuery to analyze this data. The data is too large to use Excel.

*Have you ensured your data’s integrity?*

Data integrity is the accuracy, completeness, consistency, and trustworthiness of data throughout its lifecycle. I had to change the datetime data in the original files to either just date or split between date and time. I then had to give them the right format in order to upload the tables to BigQuery. Later I discovered a way to change the data type using both R and directly in BigQuery using SQL.

*What steps have you taken to ensure that your data is clean?* 

I have double-checked the data types to make sure they are accurate and will not affect any analysis that I do.

*How can you verify that your data is clean and ready to analyze?*

I made copies of the tables I cleaned so that I can compare them to the original files.

*Have you documented your cleaning process so you can review and share those results?*

Yes, I have created a changelog.

### Key Tasks
*1. Clean the data*

Having discovered that BigQuery would not upload a table due to a problem with the datetime column, I opened the sleepDay_merged table in Excel to change that column to a date datatype so that it would be recognized by BigQuery. I renamed this table sleepDay_merged_cleaned and successfully uploaded it to my fitbit dataset in BigQuery.

Next, using the following query, I discovered that three entries looked to be duplicates.
```
SELECT
    Id,
    SleepDay,
    TotalSleepRecords,
    TotalMinutesAsleep,
    TotalTimeInBed,
    COUNT(*) AS frequency
FROM `tribal-isotope-321016.fitbit_tracker_data.sleep_day_cleaned`
GROUP BY 
    Id,
    SleepDay,
    TotalSleepRecords,
    TotalMinutesAsleep,
    TotalTimeInBed
ORDER BY frequency DESC;
```
I then checked the total number of records, returning 413.
```
SELECT  
    COUNT(*)
FROM `tribal-isotope-321016.fitbit_tracker_data.sleep_day_cleaned`;
```
Using this query, I found 410 unique records, confirming that three records are indeed duplicates.
```
SELECT 
    COUNT(*)
FROM (
    SELECT 
        DISTINCT *
    FROM `tribal-isotope-321016.fitbit_tracker_data.sleep_day_cleaned`
) AS subquery;
```
