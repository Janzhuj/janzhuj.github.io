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
library(clustertend) 
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
Number of missing value: 136534 
{% endhighlight %}

![Rplot-1](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-1.png)
