---
layout: post
title: "Exploratory Data Analysis and Predicting for Popular Film Genres"
excerpt: "EDA, Regression, Machine Learning, R"
tags: [rstats]
share: true
comments: true
---

In this post, I will investigate [movies datasets]([https://archive.ics.uci.edu/dataset/502/online+retail+ii](https://www.kaggle.com/datasets/nichen301/movie-data)) randomly sampled from Rotten Tomatos and IMDB. The purpose of the project is to identify which factors and attributes make movies popular, and develop models to predict audience responses for new movies.

## Load packages.

{% highlight r %}
library(tidyverse)
library(dlookr) 
library(gridExtra)
library(corrplot)
library(caTools)
library(caret)
library(mlbench)
library("gbm")
library(Metrics)
{% endhighlight %}

## Load the dataset and take a quick look.

{% highlight r %}
load("movies.RData")
head(movies,2)
{% endhighlight %}

{% highlight text %}
# A tibble: 2 × 32
  title        title_type genre runtime mpaa_rating studio thtr_rel_year thtr_rel_month thtr_rel_day dvd_rel_year dvd_rel_month dvd_rel_day imdb_rating
  <chr>        <fct>      <fct>   <dbl> <fct>       <fct>          <dbl>          <dbl>        <dbl>        <dbl>         <dbl>       <dbl>       <dbl>
1 Filly Brown  Feature F… Drama      80 R           Indom…          2013              4           19         2013             7          30         5.5
2 The Dish     Feature F… Drama     101 PG-13       Warne…          2001              3           14         2001             8          28         7.3
# ℹ 19 more variables: imdb_num_votes <int>, critics_rating <fct>, critics_score <dbl>, audience_rating <fct>, audience_score <dbl>,
#   best_pic_nom <fct>, best_pic_win <fct>, best_actor_win <fct>, best_actress_win <fct>, best_dir_win <fct>, top200_box <fct>, director <chr>,
#   actor1 <chr>, actor2 <chr>, actor3 <chr>, actor4 <chr>, actor5 <chr>, imdb_url <chr>, rt_url <chr>
{% endhighlight %}

The data set is comprised of 651 randomly sampled movies produced and released before 2016, Each record in the database describes characteristics of a movie.Some of these variables are only there for informational purposes and do not make any sense to include in a statistical analysis, For example information in the the actor1 through actor5 variables was used to determine whether the movie casts an actor or actress who won a best actor or actress Oscar. We select and create some new variables that might be meaningful to identify the popularity of movies.

* title_type: Type of movie (Documentary, Feature Film, TV Movie)

* genre: Genre of movie (Action & Adventure, Comedy, Documentary, Drama, Horror, Mystery & Suspense, Other)

* runtime: Runtime of movie (in minutes)

* mpaa_rating: MPAA rating of the movie (G, PG, PG-13, R, Unrated)

* imdb_rating: Rating on IMDB

* imdb_num_votes: Number of votes on IMDB

* critics_rating: Categorical variable for critics rating on Rotten Tomatoes (Certified Fresh, Fresh, Rotten)

* critics_score: Critics score on Rotten Tomatoes

* audience_rating: Categorical variable for audience rating on Rotten Tomatoes (Spilled, Upright)

* audience_score: Audience score on Rotten Tomatoes

* best_pic_win: Whether or not the movie won a best picture Oscar (no, yes)

* best_actor_win: Whether or not one of the main actors in the movie ever won an Oscar (no, yes)

* best_actress win: Whether or not one of the main actresses in the movie ever won an Oscar (no, yes)

* best_dir_win: Whether or not the director of the movie ever won an Oscar (no, yes)

* top200_box: Whether or not the movie is in the Top 200 Box Office list on BoxOfficeMojo (no, yes)

* oscar_season: New variable, Whether or not the movie is released in theaters during the oscar season.

* summer_season: New variable, Whether or not the movie is released in theaters during the summer season.

## 1. Data Preparation

{% highlight r %}
# creating new features
# Column for if movie was released during oscar season
movies <- movies %>% 
  mutate(oscar_season = as.factor(ifelse(thtr_rel_month %in% c('10', '11', '12'), 'yes', 'no')))
# Column for if movie was released during summer season
movies <- movies %>% 
  mutate(summer_season = as.factor(ifelse(thtr_rel_month %in% c('6', '7', '8'), 'yes', 'no')))

# Extracting meaningful features.
movies_new <- movies %>% select(title_type, genre, runtime, mpaa_rating, imdb_rating, imdb_num_votes, critics_rating, critics_score, audience_rating, audience_score, best_pic_win, best_actor_win, best_actress_win, best_dir_win, top200_box, oscar_season, summer_season)
{% endhighlight %}

Let's check and deal with missing values.

{% highlight r %}
cat("Number of missing value:", sum(is.na(movies_new)), "\n")
plot_na_pareto(movies_new)
{% endhighlight %}

{% highlight text %}
Number of missing value: 1 
{% endhighlight %}

![Rplot-0](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot-0.jpeg)

{% highlight r %}
movies_new<-movies_new %>%
  na.omit(runtime)
{% endhighlight %}

Split the data sets into a training Dataset and a testing Dataset,

{% highlight r %}
set.seed(101) 
sample = sample.split(movies_new$audience_score, SplitRatio = .8)
train = subset(movies_new, sample == TRUE)
test  = subset(movies_new, sample == FALSE)
{% endhighlight %}

## 2. Analyze Data
### 2.1 Descriptive Statistics

{% highlight r %}
dim(train)
# summarize feature distributions
summary(train)
# Let's also look at the data types of each feature.
sapply(train, class)
{% endhighlight %}

{% highlight text %}
        title_type                 genre        runtime       mpaa_rating   imdb_rating    imdb_num_votes           critics_rating critics_score   
 Documentary : 44   Drama             :255   Min.   : 39.0   G      : 16   Min.   :1.900   Min.   :   180   Certified Fresh: 99    Min.   :  1.00  
 Feature Film:473   Comedy            : 63   1st Qu.: 92.0   NC-17  :  2   1st Qu.:5.900   1st Qu.:  4375   Fresh          :173    1st Qu.: 32.00  
 TV Movie    :  4   Action & Adventure: 47   Median :102.0   PG     : 95   Median :6.600   Median : 15025   Rotten         :249    Median : 61.00  
                    Documentary       : 43   Mean   :105.5   PG-13  :104   Mean   :6.477   Mean   : 56558                          Mean   : 56.91  
                    Mystery & Suspense: 42   3rd Qu.:115.0   R      :266   3rd Qu.:7.300   3rd Qu.: 57251                          3rd Qu.: 82.00  
                    Horror            : 19   Max.   :267.0   Unrated: 38   Max.   :9.000   Max.   :893008                          Max.   :100.00  
                    (Other)           : 52                                                                                                         
 audience_rating audience_score  best_pic_win best_actor_win best_actress_win best_dir_win top200_box oscar_season summer_season
 Spilled:220     Min.   :11.00   no :518      no :443        no :469          no :489      no :510    no :373      no :389      
 Upright:301     1st Qu.:46.00   yes:  3      yes: 78        yes: 52          yes: 32      yes: 11    yes:148      yes:132      
                 Median :65.00                                                                                                  
                 Mean   :62.45                                                                                                  
                 3rd Qu.:80.00                                                                                                  
                 Max.   :97.00   

      title_type            genre          runtime      mpaa_rating      imdb_rating   imdb_num_votes   critics_rating    critics_score 
        "factor"         "factor"        "numeric"         "factor"        "numeric"        "integer"         "factor"        "numeric" 
 audience_rating   audience_score     best_pic_win   best_actor_win best_actress_win     best_dir_win       top200_box     oscar_season 
        "factor"        "numeric"         "factor"         "factor"         "factor"         "factor"         "factor"         "factor" 
   summer_season 
        "factor"                  
{% endhighlight %}

### 2.2 Data Visualizations

#### Correlation between categorical feature and audience score

{% highlight r %}
names <- names(Filter(is.factor,train))
plot_list <- list()

for (name in names) {
  plot <- ggplot(data = train, aes_string(x = name, y = train$audience_score, fill = name)) + 
    geom_boxplot(show.legend=FALSE) + xlab(name) + ylab('Audience Score')
  plot_list[[name]] <- plot
}
plot_grob <- arrangeGrob(grobs=plot_list, ncol=3)
grid.arrange(plot_grob)
{% endhighlight %}

![Rplot-1](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot-1.jpeg)

As we know, if median value lines from two different boxplots do not overlap, then there is a statistically significant difference between the medians. Therefore, it looks that best_actor_win, best_actress_win, best_dir_win, top200_box, oscar_season and summer_season do not seem to have much impact on audience_score.

#### Bar plot of categorical features  

{% highlight r %}
plot_list2 <- list()
for (name in names[1:6]) {
  plot <- ggplot(aes_string(x=name), data=train) + geom_bar(aes(y=100*(..count..)/sum(..count..))) + ylab('percentage')  +
    ggtitle(name) + coord_flip()
  plot_list2[[name]] <- plot
}
plot_grob2 <- arrangeGrob(grobs=plot_list2, ncol=2)
grid.arrange(plot_grob2)
{% endhighlight %}

![Rplot](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot.jpeg)

Critics_rating, audience_rating have observations spread out fairly evenly over all categories shows high variability, while  "title_type", "genre",  "mpaa_rating" and "best_pic_win"  where most observations are only in one or a handful of categories displays low variability.

#### Histogram of Numeric attributes

{% highlight r %}
names_n <- names(Filter(is.numeric,train))
hisplot_list <- list()
for (name in names_n) {
  plot <- ggplot(data = train, aes_string(x = name)) + 
    geom_histogram(aes(y=100*(..count..)/sum(..count..)), color='black', fill='white') + ylab('percentage') + ggtitle(name) 
  hisplot_list[[name]] <- plot
}
hisplot_grob <- arrangeGrob(grobs=hisplot_list, ncol=2)
grid.arrange(hisplot_grob)
{% endhighlight %}

![Rplot-4-1](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot-4-1.jpeg)

The distribution of attribute imdb_num_votes is right skewed, will be shifted by using The BoxCox transform to reduce the skew and make it more Gaussian 

#### Correlation between numerical attributes

{% highlight r %}
corr.matrix <- cor(train[names_n])
corrplot(corr.matrix, main="\n\nCorrelation Plot of numerical attributes", method="number")
{% endhighlight %}

![Rplot-4](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot-4.jpeg)

#### Summary of Ideas
There is a lot of structure in this dataset. We need to think about transforms that we could use later to better expose the structure which in turn may improve modeling accuracy. So far it would be worth trying:

* Feature selection and removing the most correlated attributes.
* standardizing the dataset to reduce the effect of differing scales and distributions.
* Box-Cox transform to see if flattening out some of the distributions improves accuracy.



## 3. Data transformation

{% highlight r %}
sapply(train[names_n],class)
train$imdb_num_votes <- as.numeric(train$imdb_num_votes)
train_num <- as.data.frame(train[names_n])
# calculate the pre-process parameters from the dataset
preprocessParams <- preProcess(train_num, method=c("center","scale","BoxCox")) # Standardize and power transform
# summarize transform parameters
print(preprocessParams)
{% endhighlight %}

{% highlight text %}
Created from 521 samples and 5 variables
Pre-processing:
  - Box-Cox transformation (5)
  - centered (5)
  - ignored (0)
  - scaled (5)
Lambda estimates for Box-Cox transformation:
-0.1, 2, 0, 0.9, 1.3
{% endhighlight %}

{% highlight r %}
# transform the dataset using the parameters
transformed <- predict(preprocessParams, train_num)
# summarize the transformed dataset
summary(transformed)
train_cat <- as.data.frame(train[names[1:6]])
# combine two datasets in r
train_trans <-cbind(train_cat, transformed)
{% endhighlight %}

Two predictors - critics score and imdb rating are highly correlated at 0.76 (collinearity). One of them will be removed from the model.

## 4: Modeling

### 4.1 Statistical regression modeling

#### Full model

{% highlight r %}
full_model <- lm(audience_score~ title_type + genre + mpaa_rating + critics_rating + audience_rating + best_pic_win + 
                   runtime + imdb_rating + imdb_num_votes, data=train_trans)
summary(full_model)
{% endhighlight %}

{% highlight text %}
Call:
lm(formula = audience_score ~ title_type + genre + mpaa_rating + 
    critics_rating + audience_rating + best_pic_win + runtime + 
    imdb_rating + imdb_num_votes, data = train_trans)

Residuals:
     Min       1Q   Median       3Q      Max 
-1.04700 -0.21807  0.02147  0.20043  1.06860 

Coefficients:
                               Estimate Std. Error t value Pr(>|t|)    
(Intercept)                    -0.69458    0.17200  -4.038 6.24e-05 ***
title_typeFeature Film          0.21918    0.14044   1.561   0.1193    
title_typeTV Movie              0.15519    0.21825   0.711   0.4774    
genreAnimation                  0.12235    0.13665   0.895   0.3710    
genreArt House & International -0.24115    0.11405  -2.114   0.0350 *  
genreComedy                     0.04441    0.06470   0.686   0.4927    
genreDocumentary                0.07563    0.14832   0.510   0.6103    
genreDrama                     -0.04539    0.05655  -0.803   0.4225    
genreHorror                    -0.12029    0.09335  -1.289   0.1982    
genreMusical & Performing Arts  0.17789    0.12421   1.432   0.1527    
genreMystery & Suspense        -0.13796    0.07315  -1.886   0.0599 .  
genreOther                      0.02849    0.10286   0.277   0.7819    
genreScience Fiction & Fantasy -0.03869    0.11998  -0.323   0.7472    
mpaa_ratingNC-17               -0.02845    0.25227  -0.113   0.9103    
mpaa_ratingPG                   0.01334    0.09859   0.135   0.8924    
mpaa_ratingPG-13               -0.01926    0.10257  -0.188   0.8512    
mpaa_ratingR                   -0.01395    0.09806  -0.142   0.8870    
mpaa_ratingUnrated              0.04496    0.11314   0.397   0.6912    
critics_ratingFresh            -0.01281    0.04598  -0.279   0.7806    
critics_ratingRotten           -0.03263    0.05068  -0.644   0.5200    
audience_ratingUpright          0.94581    0.04243  22.293  < 2e-16 ***
best_pic_winyes                -0.17528    0.19738  -0.888   0.3750    
runtime                        -0.02930    0.01715  -1.708   0.0882 .  
imdb_rating                     0.56208    0.02693  20.872  < 2e-16 ***
imdb_num_votes                 -0.00727    0.02051  -0.354   0.7231    
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

Residual standard error: 0.3276 on 496 degrees of freedom
Multiple R-squared:  0.8976,	Adjusted R-squared:  0.8927 
F-statistic: 181.2 on 24 and 496 DF,  p-value: < 2.2e-16
{% endhighlight %}


#### Obtain final model by backward stepwise selection

{% highlight r %}
newmodel <- step(full_model, direction = "backward", trace=FALSE) 
summary(newmodel)
{% endhighlight %}

{% highlight text %}
Call:
lm(formula = audience_score ~ audience_rating + runtime + imdb_rating, 
    data = train_trans)

Residuals:
     Min       1Q   Median       3Q      Max 
-0.81959 -0.23632  0.01982  0.20731  1.01312 

Coefficients:
                       Estimate Std. Error t value Pr(>|t|)    
(Intercept)            -0.56121    0.02801 -20.033   <2e-16 ***
audience_ratingUpright  0.97140    0.04160  23.352   <2e-16 ***
runtime                -0.03095    0.01490  -2.076   0.0384 *  
imdb_rating             0.54695    0.02106  25.977   <2e-16 ***
---
Signif. codes:  0 ‘***’ 0.001 ‘**’ 0.01 ‘*’ 0.05 ‘.’ 0.1 ‘ ’ 1

Residual standard error: 0.3286 on 517 degrees of freedom
Multiple R-squared:  0.8927,	Adjusted R-squared:  0.8921 
F-statistic:  1433 on 3 and 517 DF,  p-value: < 2.2e-16
{% endhighlight %}

#### Model diagnostics

{% highlight r %}
p1<-ggplot(data = newmodel, aes(x = .fitted, y = .resid)) +
  geom_point() +
  geom_hline(yintercept = 0, linetype = "dashed") +
  xlab("Fitted values") +
  ylab("Residuals")

p2<-ggplot(data = newmodel, aes(x = .resid)) +
  geom_histogram(binwidth = 0.05, fill='white', color='black') +
  xlab("Residuals")

p3<-ggplot(data = newmodel, aes(sample = .resid)) +
  stat_qq()

grid.arrange(p1,p2,p3, ncol=2)
{% endhighlight %}

![Rplot-4](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot-4.jpeg)

![Rplot-4](/figs/2022-06-25-EDA-Modeling-Movies-Popularity/Rplot-4.jpeg)







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

#### We can see a steady increase in sales over time for the retail store with a peak on September 20.

{% highlight r %}
# Observe transaction and sales patterns
# Sales by Day of Week
df %>%
  group_by(day_week) %>%
  summarise(sales = sum(sales))  %>% 
  ungroup()%>%
  hchart(type = 'column', hcaes(x = day_week, y = sales)) %>% 
  hc_yAxis(title = list(text = "Sales")) %>%  
  hc_xAxis(title = list(text = "Day of the Week")) %>% 
  hc_title(text = "Sales by Day of Week")

# Transaction by Day of Week
df %>%
  group_by(day_week) %>%
  summarise(Transaction = n_distinct(InvoiceNo))  %>% 
  ungroup() %>%
  hchart(type = 'column', hcaes(x = day_week, y = Transaction)) %>% 
  hc_yAxis(title = list(text = "Transaction")) %>%  
  hc_xAxis(title = list(text = "Day of the Week")) %>% 
  hc_title(text = "Transaction by Day of Week")

# Sales by Hour of The Day
df %>%
  group_by(hour) %>%
  summarise(sales = sum(sales)) %>% 
  ungroup() %>%
  hchart(type = 'column', hcaes(x = hour, y = sales)) %>% 
  hc_yAxis(title = list(text = "Sales")) %>%  
  hc_xAxis(title = list(text = "Hour of the Day")) %>% 
  hc_title(text = "Sales by Hour of The Day")

# Transaction by Hour of The Day
df %>%
  group_by(hour) %>%
  summarise(Transaction = n_distinct(InvoiceNo))  %>% 
  ungroup() %>%
  hchart(type = 'column', hcaes(x = hour, y = Transaction)) %>% 
  hc_yAxis(title = list(text = "Transaction")) %>%  
  hc_xAxis(title = list(text = "Hour of the Day")) %>% 
  hc_title(text = "Transaction by Hour of The Day")
{% endhighlight %}

![Rplot-14](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-14.png)
![Rplot-15](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-15.png)
![Rplot-16](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-16.png)
![Rplot-17](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-17.png)

#### By observing transaction and sales patterns, we find that the peek hours of the day in the online store are from 10 am to 15 pm, and Thursday is the busiest day.

{% highlight r %}
# Analysis products  categories by extracting and cleaning the text information
# Unique product 
products_list <- unique(df$Description)
docs <- Corpus(VectorSource(products_list))
toSpace <- content_transformer(function (x , pattern ) gsub(pattern, " ", x))
docs <- tm_map(docs, toSpace, "/")
docs <- tm_map(docs, toSpace, "@")
docs <- tm_map(docs, toSpace, "\\|")
# Convert the text to lower case
docs <- tm_map(docs, content_transformer(tolower))
# Remove numbers
docs <- tm_map(docs, removeNumbers)
# Remove English common stop words
docs <- tm_map(docs, removeWords, stopwords("english"))
# Remove your own stop word
# specify your stop words as a character vector like color, texture，size, etc.
docs <- tm_map(docs, removeWords, c("pink", "blue","red","set","white","black","sign","ivory","cover","hanging","wall","green","metal","vintage","heart", "paper", "silver", "glass","large","small","holder"))
# Remove punctuations
docs <- tm_map(docs, removePunctuation)
# Eliminate extra white spaces
docs <- tm_map(docs, stripWhitespace)

dtm <- TermDocumentMatrix(docs)
m <- as.matrix(dtm)
v <- sort(rowSums(m),decreasing=TRUE)
d <- data.frame(word = names(v),freq=v)
# Explore the 20 most common words from product Descriptions,
set.seed(1234)
wordcloud(words = d$word, freq = d$freq, min.freq = 1,
          max.words=20, random.order=FALSE, rot.per=0.35, 
          colors=brewer.pal(8, "Dark2"))
{% endhighlight %}

![Rplot-18](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-18.png)

#### The mainly products offered by the online store are decoration products, Christmas related products, bags, flowers and presents.

## Customer Segmentation &  RFM analysis & Clustering 

### Create RFM Features
Recency: how recently did customers purchase or How much time has elapsed between the last purchase date and the reference date?

Frequency: How often do customers visit or how often do they purchase?

Monetary: How much sales we get from their visit or how much do they spend when they purchase? 

{% highlight r %}
custormer_info <- df %>%
  select("InvoiceNo","InvoiceDate","CustomerID","sales") %>%
  mutate(days=as.numeric(max(df$date)-as.numeric(df$date))+1) %>%
  group_by(CustomerID) %>%
  summarise(Recency = min(days), Frequency = n_distinct(InvoiceNo), Monetary = round(sum(sales),0))  %>% 
  ungroup()
dim(custormer_info)
summary(custormer_info) 
head(custormer_info,2)
{% endhighlight %}

{% highlight text %}
[1] 4325    4

# A tibble: 2 × 4
  CustomerID Recency Frequency Monetary
       <dbl>   <dbl>     <int>    <dbl>
1      12347       3         7     4310
2      12348      76         4     1797

   CustomerID       Recency         Frequency          Monetary     
 Min.   :12347   Min.   :  1.00   Min.   :  1.000   Min.   :     3  
 1st Qu.:13816   1st Qu.: 18.00   1st Qu.:  1.000   1st Qu.:   306  
 Median :15301   Median : 51.00   Median :  2.000   Median :   664  
 Mean   :15302   Mean   : 93.46   Mean   :  4.227   Mean   :  1941  
 3rd Qu.:16778   3rd Qu.:144.00   3rd Qu.:  5.000   3rd Qu.:  1626  
 Max.   :18287   Max.   :374.00   Max.   :205.000   Max.   :278953  
{% endhighlight %}

{% highlight r %}
# Check the distributions, outliers, and correlations of RFM features
diagnose_outlier(custormer_info) %>% flextable()
plot_outlier(custormer_info, Recency, Frequency, Monetary)
check_Cor <- custormer_info %>% select("Recency", "Frequency", "Monetary")
corrplot(cor(check_Cor), 
         type = "upper", method = "ellipse", tl.cex = 0.9)
{% endhighlight %}

![Rplot-22](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-22.png)

![Rplot-23](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-23.png)

![Rplot-24](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-24.png)

![Rplot-25](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-25.png)

![Rplot-26](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-26.png)

#### Recency has negative corelation with Frequency and Monetary, while Frequency has positive correlation with Monetary.

{% highlight r %}
# Feature Scale
rfm_Scaled <- scale( select(custormer_info, -"CustomerID"))
rownames(rfm_Scaled) <- custormer_info$CustomerID
summary(rfm_Scaled)
head(rfm_Scaled,2)
{% endhighlight %}

{% highlight text %}
    Recency          Frequency          Monetary       
 Min.   :-0.9217   Min.   :-0.4270   Min.   :-0.23428  
 1st Qu.:-0.7522   1st Qu.:-0.4270   1st Qu.:-0.19764  
 Median :-0.4233   Median :-0.2947   Median :-0.15436  
 Mean   : 0.0000   Mean   : 0.0000   Mean   : 0.00000  
 3rd Qu.: 0.5038   3rd Qu.: 0.1024   3rd Qu.:-0.03804  
 Max.   : 2.7965   Max.   :26.5698   Max.   :33.49366  
 
         Recency   Frequency    Monetary
12347 -0.9017486  0.36702571  0.28648198
12348 -0.1740543 -0.02998626 -0.01736572
{% endhighlight %}

### Cluster Validation: 

#### Step one, assess the clustering tendency to see if applying clustering is suitable for the data.

{% highlight r %}
# Compute Hopkins statistic for rfm_scaled dataset
set.seed(123)
hopkins(rfm_Scaled, n = nrow(rfm_Scaled)-1)
{% endhighlight %}

#### The H value = $H 0.007978077 which is far below the threshold 0.05. Then we can reject the null hypothesis and conclude that the data set is significantly a clusterable data.

#### Step two, use Statistical testing methods to evaluate the goodness of the clustering results, choose the best algorithm and the optimal number of clusters

We start by cluster internal measures, which include the connectivity, silhouette width and Dunn index. We use clVlid function to compute simultaneously these internal measures for multiple clustering algorithms in combination with a range of cluster numbers.

{% highlight r %}
clmethods <- c("hierarchical","kmeans","pam")
intern <- clValid(rfm_Scaled, nClust = 2:6,
                  clMethods = clmethods, validation = "internal", maxitems=nrow(rfm_Scaled))
summary(intern)
{% endhighlight %}

{% highlight text %}
Clustering Methods:
 hierarchical kmeans pam 
Cluster sizes:
 2 3 4 5 6 
Validation Measures:
                                  2        3        4        5        6
                                                                       
hierarchical Connectivity    4.6496  10.6972  13.6139  20.1841  21.6841
             Dunn            0.3050   0.2483   0.2910   0.3309   0.3309
             Silhouette      0.9450   0.9381   0.9168   0.8892   0.8890
kmeans       Connectivity   16.9357  19.1258  66.0889  86.0333 113.3933
             Dunn            0.0270   0.0268   0.0004   0.0005   0.0012
             Silhouette      0.9257   0.8713   0.5939   0.6162   0.5960
pam          Connectivity   43.5409 112.5706 143.5194 230.9635 194.6619
             Dunn            0.0003   0.0006   0.0004   0.0003   0.0005
             Silhouette      0.5607   0.4615   0.4762   0.3533   0.3886

Optimal Scores:
             Score  Method       Clusters
Connectivity 4.6496 hierarchical 2       
Dunn         0.3309 hierarchical 5       
Silhouette   0.9450 hierarchical 2       
{% endhighlight %}

#### It can be seen that hierarchical clustering performs the best in each case, and the optimal number of clusters seems to be two using the Connectivity and Silhouette measures, and five using the Dunn measure.

Next, the stability measures can be computed as follow:

{% highlight r %}
stab <- clValid(rfm_Scaled, nClust = 2:6, clMethods = clmethods,
                validation = "stability", maxitems=nrow(rfm_Scaled))
# Display only optimal Scores
optimalScores(stab)
{% endhighlight %}

{% highlight text %}
           Score       Method Clusters
APN 0.0008468928 hierarchical        2
AD  0.8419747548          pam        6
ADM 0.0214144821 hierarchical        2
FOM 0.8562296447       kmeans        6
{% endhighlight %}

#### The values of APN, ADM and FOM ranges from 0 to 1, AD has a value between 0 and infinity, and smaller values are  preferred. For the APN and ADM measures, hierarchical clustering with two clusters again gives the best score. For AD measure, PAM with six clusters has the best. For FOM measure, Kmeans with six clusters has the best score.

Determine the optimal number of clusters for K-means clustering. The NbClust package provides 30 indices for choosing the best number of clusters

{% highlight r %}
nb <- NbClust(rfm_Scaled, distance = "euclidean", min.nc = 2,
              max.nc = 10, method = "kmeans")
fviz_nbclust(nb)
{% endhighlight %}

{% highlight text %}
Among all indices: 
===================
* 2 proposed  0 as the best number of clusters
* 1 proposed  1 as the best number of clusters
* 3 proposed  2 as the best number of clusters
* 7 proposed  3 as the best number of clusters
* 4 proposed  4 as the best number of clusters
* 1 proposed  5 as the best number of clusters
* 2 proposed  6 as the best number of clusters
* 2 proposed  7 as the best number of clusters
* 2 proposed  8 as the best number of clusters
* 1 proposed  9 as the best number of clusters
* 1 proposed  10 as the best number of clusters
Conclusion
=========================
* According to the majority rule, the best number of clusters is  3 .
{% endhighlight %}

![Rplot-34-1](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-34-1.png)

![Rplot-34-2](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-34-2.png)

![Rplot-33](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-33.png)

####  According to the majority rule, the best number of clusters for K-means clustering algorithm is  3.

#### Visualize cluster plots

{% highlight r %}
# Hierarchical clustering with K=2, and K=5
hc.res <- eclust(rfm_Scaled, "hclust", k = 2, hc_metric = "euclidean",hc_method = "ward.D2", graph = FALSE)
hc.res5 <- eclust(rfm_Scaled, "hclust", k = 5, hc_metric = "euclidean",hc_method = "ward.D2", graph = FALSE)
### pam clustering with K=6
pam.res <- pam(rfm_Scaled, 6, metric = "euclidean", stand = FALSE)
### Clustering: Compute k-means with k = 3 
set.seed(123)
km.res <- kmeans(rfm_Scaled, centers = 3, nstart = 35)
p1 <- fviz_cluster(hc.res, geom = "point",  ggtheme = theme_minimal())+ggtitle("Hierarchical clustering K=2")
p2 <- fviz_cluster(hc.res5, geom = "point",  ggtheme = theme_minimal())+ggtitle("Hierarchical clustering K=5")
p3 <- fviz_cluster(pam.res, geom = "point",  ggtheme = theme_minimal())+ggtitle("Pam clustering K=6")
p4 <- fviz_cluster(km.res, geom = "point", data=rfm_Scaled,ggtheme = theme_minimal())+ggtitle("k-means clustering K=3")
plot_grid(p1, p2, p3, p4, nrow=2, ncol=2)
{% endhighlight %}

![Rplot-35](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-35.png)

#### Analysis clustering results

{% highlight r %}
names(km.res)
# Lets also check the amount of customers in each cluster.
km.res$size
# centroids from model on normalized data
km.res$centers 
{% endhighlight %}

{% highlight text %}
[1]   24 1094 3207
     Recency   Frequency    Monetary
1 -0.8656131  8.57193980  9.37531267
2  1.5311005 -0.35115225 -0.17084250
3 -0.5158245  0.05563892 -0.01188207
{% endhighlight %}

{% highlight r %}
# In original customer information 
rfm_k3 <- custormer_info %>% 
  mutate(Cluster = km.res$cluster)
rfm_k3 %>% select(-"CustomerID")%>% group_by(Cluster) %>% summarise_all("mean") %>% 
  ungroup() %>% kable() %>% kable_styling()
{% endhighlight %}

![Rplot-36](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-36.png)

#### As we can see from the results, we have 3 demonstrable clusters. Cluster 1 has the newest transactions, the highest frequency, and transaction amount compared to the other 2 clusters, which contains the most valuable customers.

{% highlight r %}
# rfm features pairwise plot for each cluster
library(GGally)
rfm_df <- as.data.frame(rfm_Scaled)
rfm_df$cluster <- km.res$cluster
rfm_df$cluster <- as.character(rfm_df$cluster)
ggpairs(rfm_df, 1:3, mapping = ggplot2::aes(color = cluster, alpha = 0.5), 
        diag = list(continuous = wrap("densityDiag")), 
        lower=list(continuous = wrap("points", alpha=0.9)))

# rfm features boxplot for each cluster
f1<-ggplot(rfm_df, aes(x = cluster, y = Recency)) + 
  geom_boxplot(aes(fill = cluster))
f2<-ggplot(rfm_df, aes(x = cluster, y = Frequency)) + 
  geom_boxplot(aes(fill = cluster))
f3<-ggplot(rfm_df, aes(x = cluster, y = Monetary)) + 
  geom_boxplot(aes(fill = cluster))
plot_grid(f1, f2, f3)

# Parallel coordiante plots allow us to put each feature on seperate column and lines connecting each column
ggparcoord(data = rfm_df, columns = 1:3, groupColumn = 4, alphaLines = 0.4, title = "Parallel Coordinate Plot for the rfm Data", scale = "globalminmax", showPoints = TRUE) + theme(legend.position = "bottom")
{% endhighlight %}

![Rplot-37](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-37.png)

![Rplot-38-0](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-38.png)

![Rplot-39](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-39.png)

#### RFM analysis

{% highlight r %}
rfm_score <- rfm_table_customer(data=rfm_k3, customer_id=CustomerID, n_transactions=Frequency, recency_days=Recency,  total_revenue=Monetary,recency_bins = 5,frequency_bins =5, monetary_bins = 5)
rfm_details<-rfm_score$rfm

rfm_segments <- rfm_details %>% 
  mutate(segment = ifelse(recency_score >= 4 & frequency_score >= 4 & monetary_score >= 4, "Champion", 
                          ifelse(recency_score >= 2 & frequency_score >= 3 & monetary_score >= 3, "Loyal Customer", 
                                 ifelse(recency_score >= 3 & frequency_score <= 3 & monetary_score <= 3, "Potential Loyalist",
                                        ifelse(recency_score >= 4 & frequency_score <= 1 & monetary_score <= 1, "New Customer",
                                               ifelse((recency_score == 3 | recency_score == 4) & frequency_score <= 1 & monetary_score <= 1, "Promising",
                                                      ifelse((recency_score == 2 | recency_score == 3) & (frequency_score == 2 | frequency_score == 3) & 
                                                               (monetary_score == 2 | monetary_score == 3), "Need Attention",
                                                             ifelse((recency_score == 2 | recency_score == 3) & frequency_score <= 2 & monetary_score <= 2, "About to Sleep",
                                                                    ifelse(recency_score <= 2 & frequency_score > 2 & monetary_score > 2, "At Risk",
                                                                           ifelse(recency_score <= 1 & frequency_score >= 4 & monetary_score >= 4, "Can't lose them",
                                                                                  ifelse(recency_score <= 2 & frequency_score == 2 & monetary_score == 2, "Hibernating", "Lost")))))))))))
head(rfm_segments,5)
{% endhighlight %}

{% highlight text %}
# A tibble: 5 × 9
  customer_id recency_days transaction_count amount recency_score frequency_score monetary_score rfm_score segment       
        <dbl>        <dbl>             <dbl>  <dbl>         <int>           <int>          <int>     <dbl> <chr>         
1       12347            3                 7   4310             5               5              5       555 Champion      
2       12348           76                 4   1797             2               4              4       244 Loyal Customer
3       12349           19                 1   1758             4               1              4       414 Lost          
4       12350          311                 1    334             1               1              2       112 Lost          
5       12352           37                 6   1405             3               5              4       354 Loyal Customer
{% endhighlight %}

#### check segments threshold and size

{% highlight r %}
segments_summary<-rfm_segments %>%
  group_by(segment) %>% 
  summarise(counts=n())  %>% 
  ungroup() %>%
  arrange(-counts) 
rfm_score$threshold
head(segments_summary, 8)
{% endhighlight %}

{% highlight text %}
# A tibble: 5 × 6
  recency_lower recency_upper frequency_lower frequency_upper monetary_lower monetary_upper
          <dbl>         <dbl>           <dbl>           <dbl>          <dbl>          <dbl>
1             1            16               1               2             3            249 
2            16            34               2               3           249            487 
3            34            73               3               4           487            923 
4            73           182               4               6           923           2029.
5           182           375               6             206          2029.        278954 

# A tibble: 8 × 2
  segment            counts
  <chr>               <int>
1 Lost                 1029
2 Potential Loyalist    932
3 Champion              902
4 Loyal Customer        892
5 About to Sleep        281
6 Need Attention        173
7 At Risk                62
8 Hibernating            54
{% endhighlight %}

{% highlight r %}
fig1<-rfm_plot_median_recency(rfm_segments)
fig2<-rfm_plot_median_frequency(rfm_segments)
fig3<-rfm_plot_median_monetary(rfm_segments)
plot_grid(fig1, fig2, fig3)
{% endhighlight %}

![Rplot-40](/figs/2023-07-22-Online-Store-Customer-Segmentation/Rplot-40.png)

The souce code used to create this blog can be found [here](https://github.com/Janzhuj/Online-Store-Customer-Segmentation).
