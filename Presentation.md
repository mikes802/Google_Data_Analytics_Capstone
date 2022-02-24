# Presentation
## Share
My goal now is to make a clean, convincing presentation that I will give to Urska Srsen and Sando Mur, the founders of Bellabeat.
### Guiding Questions
*Were you able to answer the business questions?*

Our business task was:
>Identify insight into consumer use trends of smart devices that will inform Bellabeat’s marketing strategy for one Bellabeat product.

Through my analysis, I believe that I have gained insight into consumer use trends that can give the marketing team a path to a new marketing strategy.

*What story does your data tell?*

When analyzing sleep records, the data showed that there were two distinct groups, those who tracked sleep moderately or frequently, and those who rarely tracked their sleep or didn't at all. I then found that the second group, those who rarely tracked their sleep or didn't at all, had significantly more sedentary minutes on average than the first group. The first group, on the other hand, mostly fell within a fairly tight range of sedentary minutes. The data seems to show a significant relationship between sedentary minutes and sleep tracking.

*How do your findings relate to your original question?*

These findings indicate a trend in consumer use of this smart device. Those that fall within that range of sedentary minutes tend to track their sleep. Those who fall beyond that range tend not to track their sleep. These insights could help Bellabeat's marketing strategy.

*Can data visualization help you share your findings?*

Data visualization will help me tell this story in a direct, impactful way.
### Key Tasks

1. Determine the best way to share your findings.

I will create a presentation using Microsoft Powerpoint. I will be able to show my findings and visuals clearly and cleanly.

2. Create effective data visualizations.

To share the most significant value differences between Group A and Group B, the two groups I mentioned above, I think simple bar graphs with simple annotation should suffice. I downloaded the relevant data from BigQuery into cvs files and will use Excel to intially create the bar graphs. I will then copy them into Powerpoint where I can edit and enhance them visually, showing the percent change, etc. I chose to only make four bar graphs based on four values, representing the largest discrepancies between the two groups. I think this is cleaner and more compelling than using the graph I generated during my analysis. 

![Presentation Group Comparison Ave Fair Act Min](https://user-images.githubusercontent.com/99853599/155593304-4f65771e-0eba-4b7f-a17b-c32ea06e0496.png)

![Presentation Group Comparison Ave Mod Act Dist](https://user-images.githubusercontent.com/99853599/155593375-ac043c33-92af-4a84-9d72-b46e55b68b3d.png)

![Presentation Group Comparison Ave Very Act Dist](https://user-images.githubusercontent.com/99853599/155593535-5d0920b9-b3c5-4420-beb2-1e0ff4c6862a.png)

![Presentation Group Comparison Ave Sed Min](https://user-images.githubusercontent.com/99853599/155593555-cde5f4dd-8af6-49ec-b6cd-2936d376fe12.png)

The most significant discrepancy is between the average sedentary minutes between the two groups. I want to show my scatterplot graph with the trendline, but I don't think the graph itself looks weak after I paste it into Powerpoint. I will use Tableau to see if I can make a more visually pleasing scatterplot. To do this, I will first download to csv the resultant values that were returned from my last query in BigQuery. I will then upload that dataset into Tableau. I will then do the following:

1. Click on Id so that this is the dimension.
2. Drag Avg Sed Min to the Columns field so that it is my x axis.
3. Drag Number Sleep Days to the Rows field so that it is y axis.
4. Drag Id to the Detail box so that the Id's appear as scatterplots on the graph.
5. Drag Avg Sed Min to the color box so that I can adjust the color of the plot points to show a color shift from red to blue as they move up in sedentary minutes.
6. Add a trend line through the Analytics tab.

I will copy the graph to Powerpoint and add clear axis labels. The result is below:

![image](https://user-images.githubusercontent.com/99853599/155595659-c9c7b01f-ae0e-48f8-b96e-fdbaea30ceba.png)

3. Present your findings.

I will bring my findings and visuals together into my final presentation.
## Act
### Guiding Questions
*What is your final conclusion based on your analysis?*

Given the high correlation coefficient between the number of days with sleep records and average sedentary minutes, my conclusion is that this is a significant trend tht required further analysis with much larger datasets.

*How could your team and business apply your insights?*

These initial and subsequent findings could be used to drive marketing strategy. For example, the company could market sleep tracking functionality to customers who track daily activity with Bellabeat wearables and maintain an average of sedentary minutes that fall within the range discovered by this analysis. On the other hand, it's possible that promoting sleep tracking to customers who maintain high average sedentary minutes could possibly encourage them to reduce their sedentary activities, promote their health, and drive customer loyalty.   

*What next steps would you or your stakeholders take based on your findings?*

1. Gather more data to conduct a thorough analysis. With the 

*Is there additional data you could use to expand on your findings?*
