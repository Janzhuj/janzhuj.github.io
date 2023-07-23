---
layout: post
title: "Online Store Customer Segmentation"
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
df<-read_excel("Online_Retail.xlsx")
head(df,5)
str(df)
summary(df) %>% kable() %>% kable_styling()
n_distinct(df$CustomerID) 
n_distinct(df$Description)
n_distinct(df$Country)
{% endhighlight %}

The dataset consists of 541,909 rows and 8 features. Most of features have correct formats, but some features need to be converted for better analysis in the next step, like InvoiceNo and the InvoiceDate. The dataset involves 25900 transactions, 4373 customers, 4212 products and 38 country

## Data Preparation

### Data Cleaning

Let's check and deal with missing values.

{% highlight r %}
cat("Number of missing value:", sum(is.na(df)), "\n")
plot_na_pareto(df)
{% endhighlight %}

{% highlight text %}
##   Number of missing value: 136534 
{% endhighlight %}

![Rplot-1](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-1.png)


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
