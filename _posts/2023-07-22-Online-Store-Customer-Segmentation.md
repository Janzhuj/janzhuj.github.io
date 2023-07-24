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
Number of missing value: 136534 
{% endhighlight %}

![Rplot-1](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-1.png)

We find that there are 25 percent of CustomerIDs missing, and a very small percentage of Descriptions missing from the data. CustomerID can not be empty for customer segmentation analysis, and at the same time, the dataset is rich enough for serving our purpose, so we will remove the rows with missing CustomerID from the data. For the NAs on description we will replace them with an empty string value.

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

{% highlight r %}
# Check minimum UnitPrice
min(df$UnitPrice)
{% endhighlight %}

From the plots we can see that there are some negative values in the quantity, which are canceled transactions. Before we remove them, we need find out the original transactions, and then remove both negative transactions and  their positive counterparts.

{% highlight r %}
# Find canceled orders.
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
# We can Find the original Orders by selecting the same customer ID, product description, country and quantity as those in the canceled orders.
orignal_info <- order_canceled %>%
  subset(select = -c(InvoiceNo, InvoiceDate)) %>%
  mutate(Quantity  = Quantity *(-1))
# Remove the original Order that will be canceled 
df<- anti_join(df, orignal_info)
# filter all canceled orders , as well as the rows where the UnitPrice is 0. 
df <- df %>% 
  filter(Quantity > 0) %>% 
  filter(UnitPrice >0) 
# Check the outlier again
diagnose_outlier(df) %>% flextable()
plot_outlier(df, Quantity, UnitPrice)
{% endhighlight %}

![Rplot-7](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-7.png)

![Rplot-5](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-5.png)

![Rplot-6](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-6.png)

We can see now that our outlier Diagnosis Plot looks more appropriate and accurate for the analysis. We will not treat those positive outlier, since it is a custormer segmentation problem, and the outlier percentage of quantity and unitprice is around 6.38% and 8.45%, those outliers might be important affecting factors on specific segmentations.

### Feature Engineering

Create sales for each transaction

{% highlight r %}
df <- df %>%
  mutate(sales = Quantity * UnitPrice )
{% endhighlight %}

Create new features: date, time , year, month, hour, day of weak from InvoiceDate for the analysis in the next step.

{% highlight r %}
# Extract new date and time columns from InvoiceDate column
df$InvoiceDate <- as.character(df$InvoiceDate)
df$date <- sapply(df$InvoiceDate, FUN = function(x) {strsplit(x, split = '[ ]')[[1]][1]})
df$time <- sapply(df$InvoiceDate, FUN = function(x) {strsplit(x, split = '[ ]')[[1]][2]})
# Create new month, year and hour columns
df$year <- sapply(df$date, FUN = function(x) {strsplit(x, split = '[-]')[[1]][1]})
df$month <- sapply(df$date, FUN = function(x) {strsplit(x, split = '[-]')[[1]][2]})
df$hour <- sapply(df$time, FUN = function(x) {strsplit(x, split = '[:]')[[1]][1]})
# Convert date format to date type
df$date <- as.Date(df$date, "%Y-%m-%d")
# Create day of the week feature
df$day_week <- wday(df$date, label = TRUE)
{% endhighlight %}

## Exploratory Data Analysis

{% highlight r %}
# Take a look of the transactions in each county
bar <- ggplot(data = df) + 
  geom_bar(
    mapping = aes(x = Country, fill = Country), 
    show.legend = FALSE,
    width = 1
  ) + 
  theme(aspect.ratio = 1) +
  labs(x = NULL, y = NULL, title = "Transaction per Country")
bar + coord_flip()
bar + coord_polar()
{% endhighlight %}

{% highlight r %}
T_excludeUK <- df %>% 
  filter(Country != "United Kingdom")
bar <- ggplot(data = T_excludeUK) + 
  geom_bar(
    mapping = aes(x = Country, fill = Country), 
    show.legend = FALSE,
    width = 1
  ) + 
  theme(aspect.ratio = 1) +
  labs(title = "Transaction per Country Excluding UK")
bar + coord_flip()
bar + coord_polar()
{% endhighlight %}

![Rplot-8-1](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-8-1.png)

![Rplot-9-2](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-9-2.png)

#### We can see that the most of the orders come from the UK. To get a better look of the transactions that happened outside the UK, we will plot invoices excluding UK this time and lets arange it also in a descending order.

{% highlight r %}
# Take a look of the sales in each county
Sales_country <- df %>%
  group_by(Country) %>%
  summarize(sales = round(sum(sales),0), transcations=n_distinct(InvoiceNo),customers = n_distinct(CustomerID))  %>% 
  ungroup() %>%
  arrange(desc(sales))
# Sales map By Country  
Sales_country$log_sales <- round(log(Sales_country$sales),2)
highchart(type = "map") %>%
  hc_add_series_map(worldgeojson,
                    Sales_country,
                    name="sales in online store (log)",
                    value = "log_sales", joinBy = c("name","Country")) %>%
  hc_title(text = "Sales In Online Store By Country (log)") %>%
  hc_colorAxis(stops = color_stops()) %>%
  hc_legend(valueDecimals = 0, valueSuffix = "%") %>%
  hc_mapNavigation(enabled = TRUE)
{% endhighlight %}

![Rplot-10](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-10.png)


{% highlight r %}
#Top 10 sales with their transactions by country excluding UK 
Top_10_exclUK <- Sales_country %>%
  filter(Country != "United Kingdom") %>%
  top_n(10,sales)
Top_10_exclUK

highchart() %>% 
  hc_xAxis(categories = Top_10_exclUK$Country) %>% 
  hc_add_series(name = "Sales", data = Top_10_exclUK$sales) %>%
  hc_add_series(name = "Transactions", data = Top_10_exclUK$transcations) %>%
  hc_chart(
    type = "column",
    options3d = list(enabled = TRUE, beta = 15, alpha = 15)
  ) %>%
  hc_title(
    text="Top 10 Sales with their Transactions excl. UK"
  )
  {% endhighlight %}
  
![Rplot-11](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-11.png)

#### We can see that the orders of the online store come mainly from the UK, and the next top 5 countries by sales with descending order are Netherlands,EIRE (Ireland), Germany, France and Australia.

{% highlight r %}
#Top 5 sales by country excluding UK over time
top_5 <- df %>%
  filter(Country == 'Netherlands' | Country == 'EIRE' | Country == 'Germany' | Country == 'France' 
         | Country == 'Australia') %>%
  group_by(Country, date) %>%
  summarise(sales = sum(sales), transactions = n_distinct(InvoiceNo), 
                   customers = n_distinct(CustomerID)) %>%
  mutate(aveOrdVal = (round((sales / transactions),2))) %>%
  ungroup() %>%
  arrange(desc(sales))

head(top_5)
ggplot(top_5, aes(x = date, y = sales, colour = Country)) + geom_smooth(method = 'auto', se = FALSE) + 
  labs(x = ' Country', y = 'Revenue', title = 'Sales by Country over Time') + 
  theme(panel.grid.major = element_line(colour = NA),
        legend.text = element_text(colour = "skyblue4"),
        legend.title = element_text(face = "bold"),
        panel.background = element_rect(fill = NA),
        legend.key = element_rect(fill = "gray71"),
        legend.background = element_rect(fill = NA))
{% endhighlight %}
  
![Rplot-13](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-13.png)

#### We observe that the Netherlands and Australia had a sharp drop in sales after the summer of 2021.

{% highlight r %}
# Take a look of the total sales by date 
Sales_date <- df %>%
  group_by(date) %>%
  summarise(sales = sum(sales)) %>% 
  ungroup()
  
time_series <- xts(
  Sales_date$sales, order.by = Sales_date$date)

highchart(type = "stock") %>% 
  hc_add_series(time_series,
                type = "line",
                color = "green") %>%
  hc_title(text = "Sales By Date") %>% 
  hc_subtitle(text = "Sales generated from the online store") 
{% endhighlight %}

![Rplot-12](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-12.png)

