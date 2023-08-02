---
layout: post
title: "Medicare Provider Fraud Detection"
excerpt: "Classification, Python"
tags: [rstats]
share: true
comments: true
---

The US spends over $4 trillion per year on healthcare, which is largely conducted by private providers and reimbursed by insurers. Overbilling, waste, and fraud are major concerns in the healthcare system and are estimated by the US Federal Bureau of Investigation to account for 3% to 10% of overall spending. Provider fraud occurs when healthcare providers misreport their claims to receive higher payments. In this work, I use large-scale claims data from Medicare, the US federal health insurance program for elderly adults and the disabled, to perform an exploratory data analysis. My goal is to identify patterns consistent with fraud or overbilling and understand characteristics associated with high suspiciousness of fraud. My proposed approach for fraud detection is supervised and involves developing a broad range of classifiers on labeled training data to help identify potential providers who might overbill insurers. I also provide reasoning and interpretable insights into the potentially suspicious behavior of flagged providers by analyzing important features. Additionally, I apply SMOTE techniques to handle imbalanced data and improve 1-fold lift recall for the minority/fraud class. The results can be used to guide investigations and auditing of suspicious providers for both public and private health insurance systems.

## Healthcare Provider Fraud overview
Healthcare fraud and abuse take many forms. Some of the most common types that providers deceive insurers through claims procedures are listed below:

- Phantom Billing. Providers billing for services not provided.
- Unnecessary Services. Providers administering (more) tests and treatments that are not medically necessary.
- Upcoding. Providers administering more expensive tests and equipments.
- Multiple-billing. Providers multiple-billing for services rendered.
- Unbundling. Providers unbundling or billing separately for laboratory tests to get higher reimbursements.
- False price reporting. Providers charging more than peers for the same services.

## Medicare claims dataset
The dataset in this project comes from Kaggle's website - [Healthcare Provider Fraud Dection Analysis by Rohit Anand Gupta](https://www.kaggle.com/datasets/rohitrox/healthcare-provider-fraud-detection-analysis). The dataset consists of four sub-datasets, as listed below. 

![det-1](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-1.png)

## Workflow ( [see my github](https://github.com/Janzhuj/Medicare-Provider-Fraud-Detection) ) 
![det-2](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-2.png)

## Data Preporcessing 

Before discussing the extensive data analysis performed on the claims data, I would like to explain how the data was preprocessed. First, I handled missing values in the data by imputing them accordingly. I also enriched the data by creating new features. For example, in the beneficiaries dataset, the date of death was missing if a patient was alive. I created two new features: ‘if-alive’ and ‘Age’ using the date of birth and death.

Next, I label-encoded categorical data and scaled numerical data for uniform and efficient preprocessing. Here, I used RobustScaler to scale features because it is robust to outliers. The outliers in the data were kept, as they could provide important fraud indicator information and sometimes represent actual fraud transactions were being committed sometim.

Another important preprocessing step was resampling the imbalanced data to make class ratio to 1:1. Data imbalance is a typical scenario in many business problems like fraud detection, spam filtering, rare disease discovery, hardware fault detection, etc. In general, the minority/positive class is the main concern and is aimed to achieve the best results. If imbalanced data is not treated beforehand, most predictions will correspond to the majority class and treat minority class features as noise in the data and ignore them. This will result in high bias in the classifier model and degrade the model performance. Resampling data is one of the most common approaches to deal with an imbalanced dataset, including undersampling and oversampling. Here, I used two resampling techniques: 1) SMOTE, which randomly oversamples data by choosing one of K instances to interpolate new synthetic instances; and 2) SMOTE+ENN, which is a hybrid technique. ENN is an undersampling technique that removes the nearest neighbors of each majority class instance. Integrating ENN with SMOTE can clean overlapping data points for each class distributed in sample space and optimize the performance of classifier models while avoiding overfitting.

![det-3](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-3.png)

## Exploratory Data Analysis

### 1. Class Labels
Let’s take a look at the provider class dataset. It is an imbalanced dataset where the target variable, “Potential Fraud,” has 90.6% of providers not fraudulent and 9.6% of providers fraudulent. Notice, at the model evlaluation section, we will use metrics like precision, recall, F1-score rather than the accuracy metric to understand the performance of the classifiers for correctly determining which provider is fraudulent, since class distribution is high skewed, the accuracy metric is biased and not preferable to evlauate the model performance. 

![det-4](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-4.png)

### 2. Beneficiary Basic Information Study

Before we proceed, let’s take a look at our patients. We notice that the majority of our beneficiaries belong to race 1. The percentage of gender 0 is larger than that of gender 1. Fifty percent of beneficiaries fall between the ages of 68 and 82 years old, with some outliers below 47 who are disabled. The chronic disease risk scores display a bell-like shape with a slight right tail. There is a positive relationship between the mean annual reimbursement amount and the chronic disease risk scores of beneficiaries.

![det-5](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-5.png)
![det-5-1](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-5-1.png)
![det-6](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-6.png)
![det-7](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-7.png)
![det-8](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-8.png)
![det-9](/figs/2023-06-18-Medicare-Provider-Fraud-Detection/det-8.png)





{% highlight r %}
library(choroplethr)
data(df_pop_state)
head(df_pop_state)
{% endhighlight %}

{% highlight text %}
##      region   value
##1    alabama  4777326
##2     alaska   711139
##3    arizona  6410979
##4   arkansas  2916372
##5 california 37325068
##6   colorado  5042853
{% endhighlight %}

### So, what is the population of Maine?

{% highlight r %}
df_pop_state[df_pop_state$region == 'maine', ]
{% endhighlight %}

{% highlight text %}
##   region   value
##20  maine 1329084
{% endhighlight %}
 
Let's make a boxplot to see the distribution of population in the US, and where is Maine's position.

{% highlight r %}
options("scipen"=100, "digits"=4)
summary(df_pop_state$value)
boxplot(df_pop_state$value)
{% endhighlight %}

{% highlight text %}
##    Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
##  563000  1700000  4340000  6060000  6650000 37300000
{% endhighlight %}

![census-port-1](/figs/2017-03-15-Census-Data-Portland/census-port-1.png)

The median population of a state in the US is over 4.3 million, 3 times of Maine's population. And Maine's population is less than the first quartile value. This indicates that about 75% of states have a population more than Maine's, or about 25% of states have a population less than Maine's.

{% highlight r %}
state_choropleth(df_pop_state)
{% endhighlight %}

Here you go. Now We can see which states are the most/least populated.

![census-port-2](/figs/2017-03-15-Census-Data-Portland/census-port-2.png)

I think I like this color and scale better.

{% highlight r %}
state_choropleth(df_pop_state, num_colors = 1)
{% endhighlight %}

![census-port-3](/figs/2017-03-15-Census-Data-Portland/census-port-3.png)

Let me drill down to some other interesting facts. 

{% highlight r %}
data(df_state_demographics)
names(df_state_demographics)
df_state_demographics$per_capita_income[df_state_demographics$region == 'maine']
{% endhighlight %}

{% highlight text %}
## 26824
{% endhighlight %}

### The income per capita in Maine is $26824. and how is it to compare with the other states?

{% highlight r %}
df_state_demographics$value = df_state_demographics$per_capita_income
state_choropleth(df_state_demographics, num_colors=5)
{% endhighlight %} 

![census-port-4](/figs/2017-03-15-Census-Data-Portland/census-port-4.png)

Not bad, almost in the middle. 

### How about the percentage of white population?

{% highlight r %}
df_state_demographics$percent_white[df_state_demographics$region == 'maine']
df_state_demographics$value = df_state_demographics$percent_white
 state_choropleth(df_state_demographics, num_colors=5)
{% endhighlight %}

{% highlight text %}
## 94
{% endhighlight %}

![census-port-5](/figs/2017-03-15-Census-Data-Portland/census-port-5.png)

Seems Maine is among the whitest states in the nation (caucasian population at 94%). [Wikipedia](https://en.wikipedia.org/wiki/Maine) says that Maine has the highest percentage of French Americans among American states and most of the French in Maine are of Canadian Origin and they came from Quebec as immigrants between 1840 and 1930. Interesting. 

### Now, how about county, or city? 

After a quick google search, I found Portland Maine belongs to Cumberland County, the FIPS code for Cumberland County is 23005.

So the population of Cumberland County is:

{% highlight r %}
data(df_pop_county)
df_pop_county[df_pop_county$region == 23005, ]

boxplot(df_pop_county$value)
{% endhighlight %}

{% highlight text %}
##         region  value
##1180     23005   282143
{% endhighlight %}

![census-port-6](/figs/2017-03-15-Census-Data-Portland/census-port-6.png)

A little over 282,000.

It's hard to compare all counties across the US, because there are so many outliers(counties with extreme large population).

{% highlight r %}
county_choropleth(df_pop_county, num_colors=1)
{% endhighlight %}

![census-port-7](/figs/2017-03-15-Census-Data-Portland/census-port-7.png)

The national map does not tell us good story anymore, because there are so many counties. Let's zoom in. 

{% highlight r %}
county_choropleth(df_pop_county, state_zoom="maine", num_colors=4)
{% endhighlight %}

![census-port-8](/figs/2017-03-15-Census-Data-Portland/census-port-8.png)

Apparently, Cumberland is the most populated county in Maine.

### What is the median rent in Cumberland? And how is it to compare with other counties?

{% highlight r %}
data("df_county_demographics")
df_county_demographics$median_rent[df_county_demographics$region == 23005]
{% endhighlight %}

{% highlight text %}
## 847
{% endhighlight %}

{% highlight r %}
df_county_demographics$value = df_county_demographics$median_rent
county_choropleth(df_county_demographics, num_colors=1, state_zoom="maine")
{% endhighlight %}

![census-port-9](/figs/2017-03-15-Census-Data-Portland/census-port-9.png)

Oh no, Cumberland county has one of the highest median rent, people who live there must be well off!

Now let's drill down even further to the zipcode.

### What is the population of the zip code 04101?

{% highlight r %}
library(choroplethrZip)
data(df_pop_zip)
df_pop_zip[df_pop_zip$region == "04101", ]
{% endhighlight %}

{% highlight text %}
##      region value
##1067  04101  17844
{% endhighlight %}

{% highlight r %}
zip_choropleth(df_pop_zip, state_zoom="maine")
{% endhighlight %}

![census-port-10](/figs/2017-03-15-Census-Data-Portland/census-port-10.png)

It turns out, zip 04101 area is among the most populated zip areas in Maine. However, the most popuated zip area in Maine has 45087 people. 

Let's zoom in to the county level for the zipcode. 

{% highlight r %}
zip_choropleth(df_pop_zip, county_zoom=23005)
{% endhighlight %}

![census-port-11](/figs/2017-03-15-Census-Data-Portland/census-port-11.png)

Still, zip 04101 area is one of the most populated zip areas in Cumberland county, and the most populated zip area in the county has 30639 people. 

Let's explore more details on the demographics of this zip area.

{% highlight r %}
data(df_zip_demographics)
df_zip_demographics$per_capita_income[df_zip_demographics$region == "04101"]
{% endhighlight %}

{% highlight text %}
## 24560
{% endhighlight %}

The income per capita in this zip area is $24560. Let's see what that means.

{% highlight r %}
df_zip_demographics$value = df_zip_demographics$per_capita_income
zip_choropleth(df_zip_demographics, county_zoom=23005, num_colors=1)
{% endhighlight %}

![census-port-11](/figs/2017-03-15-Census-Data-Portland/census-port-11.png)

Can anyone draw any inference from this map? It seems the highest income per capita in the county is a lot more than $60K, and lowest much below $20K. 

According to above maps, I'm already jealous of people who live in Portland Maine.

The souce code used to create this blog can be found [here](https://github.com/susanli2016/Data-Analysis-with-R/blob/master/CensusDataPortland.Rmd).
