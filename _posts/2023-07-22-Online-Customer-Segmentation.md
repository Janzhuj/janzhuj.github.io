---
layout: post
title: "Online Customer Segmentation"
excerpt: "Clustering, RFM Analysis, R"
tags: [rstats]
share: true
comments: true
---

In this post, I will use clustering algorithm to perform Customer Segmentation for an e-commerce store. It will helps the business to better understand its customers, and modify its products and Marketing strategy based on the specific segments. I will be using a widely known data set from [UCI Machine Learning Repository] (https://archive.ics.uci.edu/dataset/502/online+retail+ii).  

First let’s load the required packages.

{% highlight r %}
library(tidyverse)  
library(readxl) 
library(kableExtra)
library(flextable)
library(DataExplorer) 
library(dlookr) 
library(highcharter) 
library(tm) 
library(wordcloud)
library(corrplot) 
library(viridis) 
library(xts)
library(rfm) 
library(hopkins) 
library(factoextra)
library(NbClust)  
library(clValid)
library(fpc) 
{% endhighlight %}

Load the dataset and take a quick look.

{% highlight r %}
df<-read_excel("D:/work/job/prepare/cluster/Online_Retail.xlsx")
view(df)
# Unique customers, products and country
n_distinct(df$CustomerID)
n_distinct(df$Description)
n_distinct(df$Country)
# Data Structure
str(df)
# Summary of data set
summary(df) %>% kable() %>% kable_styling()
{% endhighlight %}

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
