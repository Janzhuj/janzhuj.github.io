---
layout: post
title: "Customer Usage Data Analysis for the FitBit Fitness Tracker"
excerpt: "Data Cleaning, Exploratory Data Analysis, Time Series Analysis, R "
tags: [rstats]
share: true
comments: true
---

## Introduction

This project involves the analysis of customer usage data for a fitness tracker. Bellabeat, a company that produces FitBit fitness tracker specifically for women. The device can monitor the health vitals of its customers. Bellabeat conduct marketing analytics to expand their business. They have sampled [one-month usage data](https://www.kaggle.com/datasets/arashnic/fitbit) from thirty-three eligible Fitbit users, with their consent. Our task is to gain insights into how existing customers use their products, identify potential trends in Fitbit smart device usage, and leverage these insightful analyses to establish a new market strategy for Bellabeat. 

## Data Preparation

Load Packages and libraries

{% highlight r %}
library(tidyverse)
library(ggplot2)
library(corrplot)
library(gridExtra)
library(naniar) #check nan value
library(lessR) # pie chart
library(NbClust) 
library(factoextra)
library(tidyquant)
library(kableExtra)
{% endhighlight %}

Load the dataset and take a quick look.

{% highlight r %}
# Check the files locally using Excel and determine which file would be useful.
daily_activity <-read.csv("../FitnessTracker/dailyActivity_merged.csv")
daily_sleep <- read.csv("../FitnessTracker/sleepDay_merged.csv")
weight_log <- read.csv("../FitnessTracker/weightLogInfo_merged.csv")
# Check the number of unique Ids in the dataset
n_distinct(daily_activity$Id)
n_distinct(weight_log$Id)
n_distinct(daily_sleep$Id)
{% endhighlight %}

The distinct Id numbers in daily_activity, daily_sleep and weight_log are 33, 24, and 8 respectively. The weight_log dataset may not be useful.

{% highlight r %}
str(daily_activity)
{% endhighlight %}

{% highlight text %}
'data.frame':	940 obs. of  15 variables:
 $ Id                      : num  1503960366 1503960366 1503960366 1503960366 1503960366 ...
 $ ActivityDate            : chr  "4/12/2016" "4/13/2016" "4/14/2016" "4/15/2016" ...
 $ TotalSteps              : int  13162 10735 10460 9762 12669 9705 13019 15506 10544 9819 ...
 $ TotalDistance           : num  8.5 6.97 6.74 6.28 8.16 ...
 $ TrackerDistance         : num  8.5 6.97 6.74 6.28 8.16 ...
 $ LoggedActivitiesDistance: num  0 0 0 0 0 0 0 0 0 0 ...
 $ VeryActiveDistance      : num  1.88 1.57 2.44 2.14 2.71 ...
 $ ModeratelyActiveDistance: num  0.55 0.69 0.4 1.26 0.41 ...
 $ LightActiveDistance     : num  6.06 4.71 3.91 2.83 5.04 ...
 $ SedentaryActiveDistance : num  0 0 0 0 0 0 0 0 0 0 ...
 $ VeryActiveMinutes       : int  25 21 30 29 36 38 42 50 28 19 ...
 $ FairlyActiveMinutes     : int  13 19 11 34 10 20 16 31 12 8 ...
 $ LightlyActiveMinutes    : int  328 217 181 209 221 164 233 264 205 211 ...
 $ SedentaryMinutes        : int  728 776 1218 726 773 539 1149 775 818 838 ...
 $ Calories                : int  1985 1797 1776 1745 1863 1728 1921 2035 1786 1775 ...
{% endhighlight %}

{% highlight r %}
str(daily_sleep)
{% endhighlight %}

{% highlight text %}
'data.frame':	413 obs. of  5 variables:
 $ Id                : num  1503960366 1503960366 1503960366 1503960366 1503960366 ...
 $ SleepDay          : chr  "4/12/2016 12:00:00 AM" "4/13/2016 12:00:00 AM" "4/15/2016 12:00:00 AM" "4/16/2016 12:00:00 AM" ...
 $ TotalSleepRecords : int  1 2 1 2 1 1 1 1 1 1 ...
 $ TotalMinutesAsleep: int  327 384 412 340 700 304 360 325 361 430 ...
 $ TotalTimeInBed    : int  346 407 442 367 712 320 377 364 384 449 ...
{% endhighlight %}

#### Let's merge and prepare dataset

{% highlight r %}
# Make the date format consistent for daily_activity and data_sleep. For merging/joining dataset easier, we also need to rename both of date column names to the same  .
daily_activity <-daily_activity %>% 
  dplyr::rename(date= ActivityDate) %>%
  mutate(date= as_date(date, format= "%m/%d/%Y"))
daily_sleep <- daily_sleep %>%
  dplyr::rename(date= SleepDay) %>%
  mutate(date= as_date(date,format ="%m/%d/%Y %I:%M:%S %p" ))
# Checking the transformed tables
head(daily_activity,2)
glimpse(daily_activity)
head(daily_sleep,2)
glimpse(daily_sleep)
# Let's merge the sleep and activity table using Id and date as reference
merged_daily_activity <- merge(x = daily_activity, y = daily_sleep, by=c("Id","date"), all.x = TRUE)
# Create day of the week feature
merged_daily_activity$day_week <- wday(merged_daily_activity$date, label = TRUE)
# Checking dataset
head(merged_daily_activity,2)
{% endhighlight %}

{% highlight text %}
          Id       date TotalSteps TotalDistance TrackerDistance LoggedActivitiesDistance VeryActiveDistance ModeratelyActiveDistance
1 1503960366 2016-04-12      13162          8.50            8.50                        0               1.88                     0.55
2 1503960366 2016-04-13      10735          6.97            6.97                        0               1.57                     0.69
  LightActiveDistance SedentaryActiveDistance VeryActiveMinutes FairlyActiveMinutes LightlyActiveMinutes SedentaryMinutes Calories TotalSleepRecords
1                6.06                       0                25                  13                  328              728     1985                 1
2                4.71                       0                21                  19                  217              776     1797                 2
  TotalMinutesAsleep TotalTimeInBed day_week
1                327            346      Tue
2                384            407      Wed
{% endhighlight %}


#### Let’s check missing values

{% highlight r %}
cat("Number of missing value:", sum(is.na(merged_daily_activity)), "\n")
# plot percentage of missing values per feature 
# library(naniar)
gg_miss_var(merged_daily_activity,show_pct=TRUE)
{% endhighlight %}

![Rplot-0](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-0.jpeg)

## EDA

Correlation between numerical Variable

{% highlight r %}
# Correlation without sleep related data (drop TrackerDistance for redundant with TotalDistance, sleep related columns for not all daily_activity have an equivalent daily_sleep recorded)
data_correlation <- select(merged_daily_activity, TotalSteps:Calories, -TrackerDistance)
corrplot(cor(data_correlation))
{% endhighlight %}

![Rplot-1-1](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-1-1.jpeg)

{% highlight r %}
# Correlation with sleep realted data (drop the observations with "NA" on sleep related data)
data_corr_withsleep <- select(merged_daily_activity, TotalSteps:TotalTimeInBed, -TrackerDistance) %>% filter(!is.na(TotalTimeInBed))
corrplot(cor(data_corr_withsleep))
{% endhighlight %}

![Rplot-1-2](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-1-2.jpeg)

Based on the Correlation Plot, we can see that TotalSteps, TotalDistance (also with high correlation with TotalSteps, collinear), VeryActiveDistance, VeryActiveMinutes, and surprisingly, LightActiveDistance has a high positive correlation with Calories burned. We can summarize that as long as you walk longer distance or greater steps, it won't matter if it is intense or light activity.

{% highlight r %}
# Relationship of Calories burned with TotalDistance and TotalSteps
names_n <- c("TotalSteps", "VeryActiveDistance", "VeryActiveMinutes", "LightActiveDistance")
plt_list <- list()

for (name in names_n) {
  plt<-ggplot(data = merged_daily_activity, aes_string(x = merged_daily_activity$Calories, y = name)) + 
    geom_point(colour = "#33658A") + xlab('Calories')+ 
    geom_smooth()+
    theme_minimal()
  plt_list[[name]] <- plt
}
plt_grob <- arrangeGrob(grobs=plt_list, ncol=2)
grid.arrange(plt_grob)
{% endhighlight %}

![Rplot-1-3](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-1-3.jpeg)


#### Analyze tracker Wearing Days by Users

{% highlight r %}
user_usagedays <- merged_daily_activity %>%
  group_by(Id) %>%
  summarise(usagedays = n()) 
PieChart(usagedays, hole = 0.3, values = "%", data = user_usagedays,main = "Tracker Wearing Days by Users")     # hole = 0 pie chart
{% endhighlight %}

![Rplot-2-2](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-2-2.jpeg)

{% highlight r %}
## Data transformation
user <- user_usagedays %>% 
  group_by(usagedays) %>% # Variable to be transformed
  count() %>% 
  ungroup() %>% 
  mutate(perc = n/sum(n)) %>% 
  arrange(perc) %>%
  mutate(labels = scales::percent(perc)) 
user$usagedays<- factor(user$usagedays, levels =user$usagedays)

ggplot(data = user, aes(x = usagedays, y=n)) + geom_bar(stat="identity", fill= "#F6AE2D", colour="black") + 
  ylab('Number of users') + xlab('Number of days of wearing') + ggtitle( "Tracker Wearing Days by Users") +
  geom_text(aes(label = n), vjust = 1.5, colour = "black")+
  theme_minimal()     #  "#55DDE0", "#33658A", "#2F4858", "#F6AE2D", "#F26419", "#999999"
{% endhighlight %}

![Rplot-2](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-2.jpeg)

#### Analyze user activity behavior by weekday

{% highlight r %}
avg_weekday_activity <- merged_daily_activity %>% 
  group_by(day_week) %>% 
  summarise_at(c(Avgsteps = "TotalSteps", "Calories", "TotalMinutesAsleep"), mean,na.rm = TRUE) # see how it produces an error because of the "NA" record  

names <- names(Filter(is.numeric,avg_weekday_activity))
plt2_list <- list()

for (name in names) {
  plt<-ggplot(data = avg_weekday_activity, aes_string(x= avg_weekday_activity$day_week, y = name, fill = name)) + 
    geom_bar(stat="identity") + xlab('Day of Week')+    # geom_bar(stat="identity", fill= "#55DDE0", colour="black")
    scale_fill_gradient (low="light blue", high= "dark blue")+
    theme_minimal()+  
    theme(plot.title= element_text(hjust= 0.5,vjust= 0.8, size=16),
                            legend.position= "none")
  plt2_list[[name]] <- plt
}
plt2_grob <- arrangeGrob(grobs=plt2_list, ncol=2)
grid.arrange(plt2_grob)
{% endhighlight %}

![Rplot-3-1](/figs/2023-10-19-FitBit-Fitness-Tracker/Rplot-3-1.jpeg)

The activity Plots of the days of the week shows that Saturday recorded the most active day for our customer, followed by Tuesday. Saturday was a weekend and most people probably had more time to exercise. The curious things was that Tuesday was the second most active day. We need to dive deeper on what caused this sudden increase in activity. Customers had the highest average sleep time on Sunday, which probably from all the intense activities done on Saturday. Increasing sleep on Wednesday also coincided with the increasing activity shown on Tuesday.

#### Analyze user types
#### * method 1, Based on daily steps, we group users into 4 types: Sedentary, Lightly Active, Fairly Active, and Very Active. 
