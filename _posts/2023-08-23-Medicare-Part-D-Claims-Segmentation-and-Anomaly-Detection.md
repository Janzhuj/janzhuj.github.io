---
layout: post
title: "Medicare Part D Claims Segmentation and Anomaly Detection"
excerpt: "Medicare, EDA, Clustering, Anomaly Detection, Python"
tags: [rstats]
share: true
comments: true
---

Did you know that the United States spends over $4 trillion annually on healthcare? Unfortunately, between 3% and 10% of this spending is lost to abuse, waste, and fraud. As I researched healthcare claims data, I discovered the staggering amount spent on prescription drug claims each year. There is a crisis of drug abuse and fraud, particularly with labeled drugs such as opioids, antibiotics, long-acting opioids, and antipsychotics for the elderly. This is a serious issue that not only hurts the health and well-being of individuals and society but also leads to higher costs for care and exorbitant financial losses for individuals and organizations.

When our aim is to identify suspicious claims, we face a problem:  there is an overwhelming amount of healthcare data. It’s like finding a needle in a haystack. What we can do is to conduct claim segmentation and then identify suspicious claims. This way, we reduce the size of the search pool, cluster similar claims into groups, and make the anomaly detection task easier.  

Where can we get the data? The actual patient claim data is protected and not available to the public. However, [the Centers for Medicare and Medicaid Services (CMS)](https://data.cms.gov/search) provide aggregated claim data at the provider level. This data contains information on prescription drugs provided to Medicare beneficiaries by providers, as well as the demographics of the providers. In this case, I will use the Medicare Part D claim dataset.

Here is an overview of overall Medicare Part D Prescibers scenario in the US. We can see the number of providers in each state in 2021, and the claims percentage by drug label, brand, insurance plan, and subsidy. We can also easily identify the specialty with the most providers, claims, and costs. In addition, we have a histogram to show us the average health condition of bentificiaries.

![1](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/1.png)

![2](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/2.jpg)

![3](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/3.jpg)

![4](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/4.png)

![5](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/5.png)

![6](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/6.png)

During the exploratory data analysis section, I found that four labeled drugs - opioids, antibiotics, long-acting opioids, and antipsychotics for the elderly - accounted for 2.82% of the total drug cost. Therefore, when it comes to detecting fraudulent behavior by prescribers for labeled drugs, I selected related features to avoid interference from irrelevant information.

![7](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/7.png)

Wow, some features had high positive correlation. After some study, I decide to use  mean cost, and prescriber rate as key metrics. Now, there were no high correlated features.

![8](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/8.png)

I applied k-means clustering to segment claims and used KElbowVisualizer to identify optimum number of clusters as below.

{% highlight r %}
from yellowbrick.cluster import KElbowVisualizer
from sklearn.cluster import KMeans
print('Elbow Method to determine the number of clusters:')
plt.figure(figsize=(10,5))
Elbow_M = KElbowVisualizer(KMeans(), k=10) 
Elbow_M.fit(df_scaled)
Elbow_M.show()
{% endhighlight %}

![9](/figs/2023-08-23-Medicare-Part-D-Claims-Segmentation-and-Anomaly-Detection/9.png)

Complete code can be seen in this colab notebook. Using elbow method, I decided that 5 is the optimum number of clusters







By exploring Medicare Part D claims dataset, I conduct Exploratory data analysis, have an overview of overall Medicare Part D Prescibers scenario in the US, to perform Providers Segmentation and Anomaly Detection. It’s like finding a needle in a haystack. More precisely, it’s like finding a faulty needle in a pool of needles. Enter Graph Analysis to the rescue. Using Tableau dashboards, I will show how we can filter the providers and identify suspicious ones. This way, we reduce the size of the search pool, so that finding a faulty needle becomes easy. The souce code used to create this blog can be found here.o se

I cleaned the dataset, and conduct Exploratory data analysis. An overview of overall healthcare provider/prescriber scenario in the US; the charts also act as filters for other charts, so we can see the sickness level of patients of California going to see a dentist in the bottom right bar chart using the two charts above.




{% highlight r %}
url_causes <- "https://ibm.box.com/shared/static/souzhfxe3up2hrh23phciz18pznbtqxp.csv"

df <- read.csv(url_causes)
unique(df$Cause)
sum(df$Deaths)
unique(df$Year)
unique(df$Gender)
unique(df$Age)
df[!complete.cases(df), ]
{% endhighlight %}

This mortality data contains 51 causes and 6540835 deaths for the year 2005, 2010 and 2015. The gender are male and female, and the age from 0 to 100. And there is no missing value in the dateset.

### So, what are the top causes of death in the United States?


{% highlight r %}
library(ggplot2)
library(ggthemes)
ggplot(aes(x = reorder(Cause,-Deaths), y = Deaths), data = df) + 
  geom_bar(stat = 'identity') +
  ylab('Total Deaths') +
  xlab('') +
  ggtitle('Causes of Deaths') +
  theme_tufte() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
{% endhighlight %}


Heart disease and cancer are far away the most important causes of death in the US. Let?s take these top 10 causes of death and make a new data frame for some plotting, as follows:


{% highlight r%}
library(dplyr)
cause_group <- group_by(df, Cause)
df.death_by_cause <- summarize(cause_group, 
                               sum = sum(Deaths))
df.death_by_cause <- arrange(df.death_by_cause, desc(sum))
top_10 <- head(df.death_by_cause, 10)
top_10
ggplot(aes(x = reorder(Cause, sum), y = sum), data = top_10) +
  geom_bar(stat = 'identity') +
  theme_tufte() +
  theme(axis.text = element_text(size = 12, face = 'bold')) +
  coord_flip() +
  xlab('') +
  ylab('Total Deaths') +
  ggtitle("Top 10 Causes of Death")
{% endhighlight %}


{% highlight text %}
## A tibble: 10 × 2
##                                         Cause     sum
##                                        <fctr>   <int>
## 1                            Diseases of heart 1883528
## 2                          Malignant neoplasms 1729960
## 3           Chronic lower respiratory diseases  424042
## 4                     Cerebrovascular diseases  413364
## 5           Accidents (unintentional injuries)  385148
## 6                          Alzheimer's disease  265653
## 7                            Diabetes mellitus  223721
## 8                      Influenza and pneumonia  170153
## 9  Nephritis, nephrotic syndrome and nephrosis  144333
## 10             Intentional self-harm (suicide)  115175
{% endhighlight %}

![Rplot-2](/figs/2017-03-25-Causes-of-Death/Rplot-2.png)

Heart desease remains the leading cause of death in the US, followed by cancer(Malignant neoplasms), Chronic lower respiratory desease takes the third place but by a wide margin.

### Are men more likely to die than women? Or vice versa? Does it depend on cause of death or age?

{% highlight r %}
df_top_10 <- df[df$Cause %in% unique(top_10$Cause), ]

ggplot(aes(x = reorder(Cause, Deaths), y = Deaths, fill = Gender), data = df_top_10) +
  geom_bar(stat = 'identity', position = position_dodge()) +
  theme_tufte() +
  coord_flip() +
  ggtitle("Top 10 Causes of Death") +
  facet_wrap(~Year) +
  theme_fivethirtyeight() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) 
{% endhighlight %}

![Rplot-3](/figs/2017-03-25-Causes-of-Death/Rplot-3.png)


The picture does not change much when we compare 2005, 2010 and 2015. And more Ameican women than men dying for heart disease. Also Stroke(cerebrovascular diseases), Alzheimer's disease and Influenza and pneumonia seem affect more women than men. 


{% highlight r %}
ggplot(aes(x = Age, y = Deaths, color = Cause), 
       data = df_top_10[df_top_10$Gender == 'M',]) +
  geom_smooth(method = 'loess') +
  ggtitle('Causes of Death by Age for Male')
{% endhighlight %}

![Rplot-4](/figs/2017-03-25-Causes-of-Death/Rplot-4.png)

In general, cancer affects male a little bit earlier than heart disease, at peak around 70 years old, heart disease affects more people at older age peak around 75 years old. 

Only two diseases affect more yonger people than older people - Accidents and Suicide.  

{% highlight r %}
ggplot(aes(x = Age, y = Deaths, color = Cause), 
       data = df_top_10[df_top_10$Gender == 'F',]) +
  geom_smooth(method = 'loess') +
  ggtitle('Causes of Death by Age for Female')
{% endhighlight %}

![Rplot-5](/figs/2017-03-25-Causes-of-Death/Rplot-5.png)

Female developes heart disease more than 10 years later than male, at peak almost at 100 years old. Men appear more vulnerable to heart disease than the inaptly-named 'weaker sex'. As a result, they succumb earlier.

### What the trend look like over time?

A stacked area plot may help to display the trend and development of each cause of deaths over time. 

{% highlight r %}
ggplot(aes(x = Age, y = Deaths, color = Cause, fill = Cause), 
       data = df_top_10) +
  geom_area() +
  ggtitle('Causes of Death by Age and Gender') +
  facet_wrap(~Gender) +
  theme_solarized()
{% endhighlight %}

![Rplot-6](/figs/2017-03-25-Causes-of-Death/Rplot-6.png)

To see the trend over time, I downloaded a new dataset from [Centers for Disease Control and Prevention(CDC)](https://wonder.cdc.gov/ucd-icd10.html) and did some cleaning up. This dataset contains deaths, population, crude rate per 100,000 nationwide from 1999 to 2015.


{% highlight r %}
library(xlsx)
mydata <- read.xlsx("mydata.xlsx", 1)
mydata$NA. <- NULL

ggplot(aes(x = Year, y = Deaths), data = mydata) +
        geom_line(size = 2.5, alpha = 0.7, color = "mediumseagreen") +
        geom_point(size = 0.5) + xlab("Year") + ylab("Number of deaths") +
        ggtitle("Deaths in the US")
{% endhighlight %}

![Rplot-7](/figs/2017-03-25-Causes-of-Death/Rplot-7.png)

Oh no! This is terrible. The number of death are going up! But the population of US has been growing. We should look at the per capita number of deaths. These things are typically measured per 100,000 population.

{% highlight r %}
ggplot(aes(x = Year, y = Crude_Rate_per_100K), data = mydata) +
        geom_line(size = 2.5, alpha = 0.7, color = "mediumseagreen") +
        geom_point(size = 0.5) + xlab("Year") + ylab("Number of deaths per 100,000 population") +
        ggtitle("Deaths in the US")
{% endhighlight %}

![Rplot-8](/figs/2017-03-25-Causes-of-Death/Rplot-8.png)

## The End

The death rate had been declining for years, an effect of improvement for health and disease control and technology. Until a few years ago, it has been raising in the recent years. If we want more accurate data, we will need the age-adjusted death rate for the total population because [the birth rate in the US drops to the lowest point](http://www.cnn.com/2016/08/11/health/us-lowest-fertility-rate/). The population 10 years ago was yonger than the population now. 

That's all for now, next time, probably state by state.

The souce code used to create this blog can be found here.
