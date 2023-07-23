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

We find that there are 25 percent of CustomerIDs missing, and a very small percentage of Descriptions missing from the data. CustomerID can not be empty for customer segmentation analysis, and at the same time, the dataset is rich enough for serving our purpose, so we will remove the rows with missing CustomerID from the data.
For the NAs on description we will replace them with an empty string value.

{% highlight r %}
df<-df %>%
  na.omit(CustomerID)
df$Description<-replace_na(df$Description, "N/A")
{% endhighlight %}

Check and deal with outlines

{% highlight r %}
diagnose_outlier(df) %>% flextable()
plot_outlier(df,Quantity, UnitPrice)
{% endhighlight %}

![Rplot-2](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-2.png)

![Rplot-3](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-3.png)

![Rplot-4](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-4.png)

Check minimum UnitPrice
{% highlight r %}
min(df$UnitPrice)
{% endhighlight %}

From the plots we can see that there are some negative values in the quantity, which are canceled transactions. Before we remove them, we need find out the original transactions, and remove both negative transactions and  their positive counterparts.

Check canceled orders. 
{% highlight r %}
order_canceled <- df %>%
  filter(Quantity<0) %>%
  arrange(Quantity)
head(order_canceled, 2)
df %>% 
  filter(CustomerID == 16446)
{% endhighlight %}

{% highlight text %}
# A tibble: 2 × 8
  InvoiceNo StockCode Description                    Quantity InvoiceDate         UnitPrice CustomerID Country       
  <chr>     <chr>     <chr>                             <dbl> <dttm>                  <dbl>      <dbl> <chr>         
1 C581484   23843     PAPER CRAFT , LITTLE BIRDIE      -80995 2011-12-09 09:27:00      2.08      16446 United Kingdom
2 C541433   23166     MEDIUM CERAMIC TOP STORAGE JAR   -74215 2011-01-18 10:17:00      1.04      12346 United Kingdom

# A tibble: 4 × 8
  InvoiceNo StockCode Description                 Quantity InvoiceDate         UnitPrice CustomerID Country       
  <chr>     <chr>     <chr>                          <dbl> <dttm>                  <dbl>      <dbl> <chr>         
1 553573    22980     PANTRY SCRUBBING BRUSH             1 2011-05-18 09:52:00      1.65      16446 United Kingdom
2 553573    22982     PANTRY PASTRY BRUSH                1 2011-05-18 09:52:00      1.25      16446 United Kingdom
3 581483    23843     PAPER CRAFT , LITTLE BIRDIE    80995 2011-12-09 09:15:00      2.08      16446 United Kingdom
4 C581484   23843     PAPER CRAFT , LITTLE BIRDIE   -80995 2011-12-09 09:27:00      2.08      16446 United Kingdom
{% endhighlight %}


{% highlight r%}

{% endhighlight %}
