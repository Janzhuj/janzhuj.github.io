---
layout: post
title: "Online Customer Segmentation"
excerpt: "Clustering, RFM Analysis, R"
tags: [rstats]
share: true
comments: true
---

In this post, I will use clustering algorithm to perform Customer Segmentation for an e-commerce store. It will helps the business to better understand its customers, and modify its products and Marketing strategy
based on the specific segments. I will be using a widely known data set from UCI Machine Learning Repository (https://archive.ics.uci.edu/dataset/502/online+retail+ii).  

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
