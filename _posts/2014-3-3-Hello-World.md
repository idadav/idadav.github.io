---
layout: post
title: Data exploration on the Madrid AirBnB dataset
---
As a project for my MSc in Big Data, one of my assignments were to do some data exploration and statistical modelling on a dataset of my choice.

I decided to use the Madrid Airbnb dataset which consists of listings, details about listings, prices, reviews, ratings amongst others. Airbnb started as a small tech start-up where private people could list spare rooms and/or homes for rent to earn some extra money. As it's popularity grew, this business model changed and Airbnb are now facing scrutiny in many cities due to a lot of apartments being purchased for the sole purpose of being rented at Airbnb, driving up housing prices. I want to look at this dataset and see the level of density of Airbnb listings in different areas of Madrid, the price levels, how apartments are being rated depending on how many apartment the host owns etc.

The questions I am asking myself before modeling and visualising the results are:

0. does price depend on room type and/or location?
1. are you more likely to be a super_host if you have a lower price?
2. does the amount of houses a host has listed impact reviews and price?
3. what is the relationship between deposit and cleaning fees with reviews per month as a measure of demand?
I will also run a linear regression model to anayse which variables predict price.

The above can be evaluated using the following variables (list not exhaustive):

price
room_type
neighbourhood
host_listings_count
reviews_per_month
reviews_score_rating

```{r - Load necessary libraries}
# Loading all libraries we need
library(tidyverse)
library(broom)
library(gridExtra)
library(ggmap)
library(maps)
library(ggpubr)
library(car)
library(gplots)
library(rworldmap)
library(dotCall64)
library(MASS)
library(caret)
```


```{r - Import, Read and Clean Data}
# import dataset
airbnb = read.csv("listings.csv")

# get a first overview of the dataset
glimpse(airbnb)
str(airbnb)

# removing unnecessary columns
airbnb$notes = NULL
airbnb$space = NULL
airbnb$listing_url = NULL
airbnb$neighborhood_overview = NULL
airbnb$transit = NULL
airbnb$summary = NULL
airbnb$interaction = NULL
airbnb$house_rules = NULL
airbnb$host_url = NULL
airbnb$picture_url = NULL
airbnb$thumbnail_url = NULL
airbnb$medium_url = NULL
airbnb$name = NULL
airbnb$description = NULL
airbnb$host_about = NULL
airbnb$host_acceptance_rate = NULL
airbnb$host_picture_url = NULL
airbnb$host_thumbnail_url = NULL
airbnb$host_verifications = NULL
airbnb$jurisdiction_names = NULL
airbnb$require_guest_phone_verification = NULL
airbnb$require_guest_profile_picture = NULL
airbnb$license = NULL
airbnb$country_code = NULL
airbnb$country = NULL
airbnb$experiences_offered = NULL
airbnb$last_scraped = NULL
airbnb$xl_picture_url = NULL
airbnb$access = NULL
airbnb$city = NULL
airbnb$state = NULL
airbnb$smart_location = NULL
airbnb$has_availability = NULL
airbnb$square_feet = NULL
```

Removing dollar sign so we can work with the data
```{r - Clean data_2}
airbnb$price = as.numeric(gsub("\\$", "", airbnb$price))
airbnb$weekly_price = as.numeric(gsub("\\$", "", airbnb$weekly_price))
airbnb$monthly_price = as.numeric(gsub("\\$", "", airbnb$monthly_price))
airbnb$security_deposit = as.numeric(gsub("\\$", "", airbnb$security_deposit))
airbnb$cleaning_fee = as.numeric(gsub("\\$", "", airbnb$cleaning_fee))
airbnb$extra_people = as.numeric(gsub("\\$", "", airbnb$extra_people))

```


>QUESTION 1: 
does price depend on room type and/or location

```{r - Map visualization on whole dataset}
# Creating a map to visualies room types in Madrid
map = get_map(location ='Madrid', zoom = 13)

# mapping location and room_type with whole dataset
ggmap(map) + 
  geom_point(aes(x = longitude, y = latitude, size = room_type, colour = room_type),   data=airbnb, alpha=0.5)

```

As we can see there are too many points, we can not see groups or different areas. That's why we are going to take a random sample of 5% for a clearer visualtisation. 


```{r - Map visualization on Sample data}
# creating a 0.05 sample
airbnb_random5 = airbnb %>%
  sample_frac(0.05, replace = TRUE)

# mapping location + room_type with 0.05 sample
ggmap(map) + 
  geom_point(aes(x = longitude, y = latitude, colour = room_type, size = room_type),   data=airbnb_random5, alpha=0.5)

```

In central madrid it's more whole apartments and in the suburbs people are renting out rooms in their own apartments. 

We will rank the neighbourhoods by mean price but we will also check the median value and the normality of the prices.
```{r - Summary stats on price ~ neighbourhood}
# calculation mean price / neighbourhood 
avg_p = airbnb %>%
  group_by(neighbourhood) %>%
  summarise(avg_price = mean(price, na.rm = TRUE))

as.data.frame(avg_p)

# ranking mean price / neighbourhood
avg_p[order(avg_p$avg_price, decreasing = TRUE), c(1,2)]

# calculating median price / neighbourhood
median_p = airbnb %>%
  group_by(neighbourhood) %>%
  summarise(median_price = median(price, na.rm = TRUE))

as.data.frame(median_p)

# ranking median price / neighbourhood
median_p[order(median_p$median_price, decreasing = TRUE), c(1,2)]
```

In terms of mean, Recoletos, Castellana and Chameberi are the most expensive neighbourhoods. The same holds for median, except that Chamberi falls and Jeronimos takes place 3. This tells us that although Chamberi is expensive on average, Chamberi probably also houses a lot of cheap rooms for rent.


```{r - Check normality of Price}
# plotting price levels
ggplot(data=airbnb, aes(x=price)) + 
  geom_histogram(aes(colour=I("black")), binwidth=10) +
  theme_minimal()  + labs(title="Euro price per night in Madrid")

shapiro.test(airbnb_random5$price)
```

Right tail, this is due to a great amount of houses available in Madrid. We can not forget that Airbnb was made to rent private houses or rooms for little money.

We used the random sample cause the sample must be between 3 and 5000. A we can see we reject normality.

Lets then look at room_type vs price
```{r - Price~Room_type}
# plotting price depending on room type
ggplot(data=airbnb, aes(x=price)) + 
  geom_histogram(aes(colour=I("grey"), fill = room_type), binwidth = 40, position =   "Dodge") + 
  theme_minimal() + 
  labs(x="Price in Euros", title="Price per home type") + 
  xlim(0,1000)

```

Next, we conduct an ANOVA test to check if the averages are different. Our hypothesis follows:
$$
\left\{ \begin{eqnarray*}
H_0: &\mu_{pHOME} = \mu_{pPRIVATE} = \mu_{pSHARED}  \\
H_a: & \text{ at least two means are different} 
\end{eqnarray*} \right.
$$

```{r - ANOVA on Price~Room_type }
# ANOVA test to check if averages in price are different
aov1 = aov(data=airbnb, price ~ room_type)
summary(aov1)

# post hoc analysis
TukeyHSD(aov1)
plot(TukeyHSD(aov1))

# plotting for visualisation 
med_price = median(airbnb$price)
mean_price = mean(airbnb$price)

ggplot(data=airbnb, aes(x=room_type, y=price)) + 
  geom_boxplot() + 
  geom_text(data=airbnb, aes(x=room_type, y=mean_price, label=mean_price)) + 
  theme_minimal()

ggplot(data=airbnb, aes(sample=scale(price))) + 
  stat_qq() + 
  facet_wrap(~room_type, nrow=3) + 
  geom_abline(intercept = 0, slope = 1) + 
  theme_minimal() + 
  labs(title="Theoretical vs Actual Price", x="Theoretical value")
```

Conclusion: we can reject the null hypothesis that the average price per room type are similar as the p-value is below alpha (0.05). 


>QUESTION 2: Are you more likely to be a super_host if you have a lower price?

Before conducting any analysis, we'll take a look at the distribution of Price with a very heavy right tail to see the impact of removing the outliers (the small number of high-value listings)
```{r - Outlier test for Price & Removing extra factor in superhost}
#Identify NA values and cleaning them 
summary(airbnb$host_is_superhost)

#superhost initially had 3 levels with the third level being " ". The NA values and unused factors were removed. 
airbnb_sh <- airbnb %>%
  filter(host_is_superhost == "t" | host_is_superhost == "f") %>%
  droplevels()

#Summary stats with outliers
summary(airbnb_sh$price)

#Visualize Price using density plot, histogram and boxplot with all data
ggplot(data=airbnb_sh) + 
  geom_density(aes(x=price, colour=I("blue")), size=1) +
  theme_minimal() +
  labs(x="Price", title="Density plot of Price")

ggplot(data=airbnb_sh) + 
  geom_histogram(aes(x=price, colour=I("white")), size=1) +
  theme_minimal() +
  labs(x="Price", title="Histogram of Price")

c1 <- ggplot(data=airbnb_sh) + 
  geom_boxplot(aes(x = host_is_superhost,y= price, colour=I("black")), size=1) +
  theme_minimal() +
  labs(x="Price", title="Histogram of Price")

# Removing outliers more than 3rd IQR based on summary stats
airbnb_3Q <- airbnb_sh %>%
  filter(price <= 80 & !is.na(price))

summary(airbnb_3Q$price)

ggplot(data=airbnb_3Q) + 
  geom_density(aes(x=price, colour=I("blue")), size=1) +
  theme_minimal() +
  labs(x="Price", title="Density plot of Price (3rd Quantile)")

ggplot(data=airbnb_3Q) + 
  geom_histogram(aes(x=price, colour=I("white")), size=1) +
  theme_minimal() +
  labs(x="Price", title="Histogram of Price (3rd Quantile)")

c2 <- ggplot(data=airbnb_3Q) + 
  geom_boxplot(aes(x = host_is_superhost,y= price, colour=I("black")), size=1) +
  theme_minimal() +
  labs(x="Price", title="Histogram of Price (3rd Quantile)")

#box-plot of price with whole dataset and without outliers
grid.arrange(c1,c2)

#impact of removing outliers
cat("Number of values removed:",length(airbnb_sh$price) - length(airbnb_3Q$price))

```

While removing the outlying variables does present a much more visually pleasing visualization of the dataset, it introduces bias in the data as more than 3200 data points have been removed from the data and this could provide some valuable insight with further tests. As a reult, we have decided to keep the outling variables for further tests 

What does the price and number of reviews look like when compared with a super_host status? 
```{r - Visualizing price/no_of_reviews~superhost}

#Standardize price and no_of reviews before graphing
zprice = scale(airbnb_sh$price)
zreviewcount = scale(airbnb_sh$number_of_reviews)

ggplot(data = airbnb_sh) +
  geom_point( aes(y= zreviewcount, x = zprice, alpha = 0.2)) + 
  facet_grid(host_is_superhost~.) +
  theme_minimal()

ggplot(data = airbnb_sh) +
  geom_point( aes(y= review_scores_rating, x = zprice, colour = neighbourhood_group_cleansed, alpha = 0.2)) +
  facet_grid(host_is_superhost~.) +
  theme_minimal()

#Plotting individual factors against superhost
ggplot(data=airbnb_sh) +
  geom_histogram(aes(x=price, y=..density..), colour =I("White")) +
  facet_grid(host_is_superhost~.) +
  theme_minimal()

ggplot(data=airbnb_sh) +
  geom_histogram(aes(x=number_of_reviews, y=..density..), colour =I("White")) +
  facet_grid(host_is_superhost~.) +
  theme_minimal()

ggplot(data=airbnb_sh) +
  geom_histogram(aes(x=review_scores_rating, y=..density..), colour =I("White")) +
  facet_grid(host_is_superhost~.) +
  theme_minimal()
```

```{r - t-test on price/number of reviews/review scores rating ~ Superhost}

t.test(airbnb_sh$price[airbnb_sh$host_is_superhost=='t'], airbnb_sh$price[airbnb_sh$host_is_superhost=='f'], alternative = "greater")

t.test(airbnb_sh$number_of_reviews[airbnb_sh$host_is_superhost=='t'], airbnb_sh$number_of_reviews[airbnb_sh$host_is_superhost=='f'], alternative = "greater")

t.test(airbnb_sh$review_scores_rating[airbnb_sh$host_is_superhost=='t'], airbnb_sh$review_scores_rating[airbnb_sh$host_is_superhost=='f'], alternative = "greater")
```

With all 3 t-tests, the p-value is much lower than the alpha value of 0.05 which means that there is enough evidence to show that the price, number of reviews and reviews scores ratings are greater when the listing is from a superhost. 


> QUESTION 3: does the amount of houses a host has listed impact reviews and price?

```{r - Summary stats and Scaling}
# First overlook of average and median values for the variables that we will be using
airbnb %>%
  summarise(
    avg_houses = mean(host_listings_count, na.rm=TRUE),
    med_houses = median(host_listings_count, na.rm=TRUE),
    avg_rating = mean(review_scores_rating, na.rm=TRUE),
    med_rating = median(review_scores_rating, na.rm=TRUE),
    avg_price = mean(price, na.rm=TRUE),
    med_price = median(price, na.rm=TRUE)
    )

# Scaling for comparability in price vs rating
zprice = scale(airbnb$price)
zrating = scale(airbnb$review_scores_rating)

# Z-scores of price vs rating
ggplot(data=airbnb) + 
  geom_density(aes(x=zprice, colour=I("blue")), size=1) +
  geom_density(aes(x=zrating, colour=I("red")), size=1) +
  theme_minimal() +
  labs(x=" Z-scores of price and ratings", title="price vs rating")

# binning host_listings_count for ANOVA analysis
airbnb_hosts = airbnb %>%
  filter(!is.na(price)) %>%
  filter(!is.na(review_scores_rating)) %>%
  mutate(level = ifelse(host_listings_count == 1, "low", 
                    ifelse(host_listings_count >= 2 & host_listings_count <= 3, "med", "high"))) 

# With these new factor levels, lets plot some initial histograms to see if we can visualise any differences in avg price and/or rating
ggplot(data=airbnb_hosts, aes(x=price)) + 
  geom_histogram(aes(fill=level, colour=I("white")), position="dodge", binwidth=40) +
  theme_minimal() + labs(x="Euro price", title="price per night / houses owned")

ggplot(data=airbnb_hosts, aes(x=review_scores_rating)) +
  geom_histogram(aes(fill=level, colour=I("white")), position="dodge", binwidth=5) +
  theme_minimal() + labs(x="ratings score", title="ratings / houses owned")
  
```

From looking at the above, it does indeed look like the hosts with a high amount of houses listed (>3) have a higher price but lower ratings (they are more previalent in the higher price areas and lower rating areas compared to med and low owners). Lets see if we can confirm these differences with statistical tests.

We run an anova test to see if there is enough evidence to claim that at least two means in price vs amount of houses owned are different. Our hypotheses follows:
$$
\left\{ \begin{eqnarray*}
H_0: &\mu_{pLOW} = \mu_{pMED} = \mu_{pHIGH}  \\
H_a: & \text{ at least two means are different} 
\end{eqnarray*} \right.
$$

```{r - ANOVA on Price~Level}
# anova test for price
price_a = aov(data=airbnb_hosts, price ~ level)
summary(price_a)

# post hoc analysis
TukeyHSD(price_a)
plot(TukeyHSD(price_a))
plot(price_a, 1)
plotmeans(price ~ level, data = airbnb_hosts, xlab = "Level", ylab = "Price", main="Mean Plot with 95% CI", barwidth=3, barcol="red")

```

Conclusion: since p-value is less than alpha, so we REJECT H0. There is enough evidence to claim that the average price between the amount of houses owned are different. Our post hoc analysis confirms this, and it looks from the 95% CI plot that the hosts with a high number of houses are driving this (with higher prices).

Next, we run the same anova test but this time for the review ratings:
$$
\left\{ \begin{eqnarray*}
H_0: &\mu_{rLOW} = \mu_{rMED} = \mu_{rHIGH}  \\
H_a: & \text{ at least two means are different} 
\end{eqnarray*} \right.
$$

```{r - ANOVA review_scores~Level}
# anova test for review ratings
rating_a = aov(data=airbnb_hosts, review_scores_rating ~ level)
summary(rating_a)

# post hoc analysis
TukeyHSD(rating_a)
plot(TukeyHSD(rating_a))
plot(rating_a, 1)
plotmeans(review_scores_rating ~ level, data = airbnb_hosts, xlab = "Level", ylab = "Rating", main="Mean Plot with 95% CI", barwidth = 3, barcol="red")
```

Conclusion: since p-value is less than alpha, so we REJECT H0. There is enough evidence to claim that the average review score between the amount of houses owned are different. We can see verify this with our post hoc analysis. Again, we can see from the 95% CI plot that the hosts with a high amount of houses might be driving this, as they appear to have a lower rating.

Assessing normality:
```{r - Assessing normality of Price,Ratings}
# normality for price
ggplot(data=airbnb_hosts, aes(sample=scale(price))) + stat_qq() + facet_wrap(~ level, nrow=3) + geom_abline(intercept=0, slope=1)

# normality for ratings
ggplot(data=airbnb_hosts, aes(sample=scale(review_scores_rating))) + stat_qq() + facet_wrap(~ level, nrow=3) + geom_abline(intercept=0, slope=1)

```

Overall conclusion of amount of houses owned is that there are differences in price and average ratings depending on how many houses the host owns. It appears to be driven by the hosts having a high amount of houses. Maybe this could be due to companies listing their houses on airbnb and thus charging more and giving less of a personal service expected when renting someones home/room in someones home at airbnb, and consequntly receiving lower ratings.


> QUESTION 4: what is the relationship between deposit and cleaning fees with reviews per month as a measure of demand?

Important to keep in mind for this part is that according to the description of the dataset, amount of reviews account for circa 70% of all bookings. So by using amount of reviews, we can ~ estimate number of bookings.
```{r - Visualization of variables}
# Lets first visualise the variables we will use
rpm = ggplot(data=airbnb, aes(x=reviews_per_month)) +
  geom_histogram(aes(colour=I("black")), binwidth = 1) +
  theme_minimal() +
  labs(title="Madrid reviews/month")

secd = ggplot(data=airbnb, aes(x=security_deposit)) + 
  geom_histogram(aes(colour=I("black")), binwidth = 30) +
  theme_minimal() +
  labs(title="Madrid deposit")

cf = ggplot(data=airbnb, aes(x=cleaning_fee)) + 
  geom_histogram(aes(colour=I("black"))) +
  theme_minimal() +
  labs(title="Madrid cleaning fee")

grid.arrange(rpm, secd, cf, nrow=1)

# We check if the NA values are 0. 
sample_clean = dplyr::select(airbnb, cleaning_fee)
sample_clean %>%
  arrange(cleaning_fee) %>%
  head(10)

sample_deposit = dplyr::select(airbnb, security_deposit)
sample_deposit %>%
  arrange(security_deposit) %>%
  head(10)
```

We can see that there are no flats without cleaning fee or depoosit which means the NA are 0.

```{r}
# so now we turn the Na into 0 to work better with the data
cleaning = airbnb %>%
  mutate(cleaning = ifelse(is.na(cleaning_fee), 0, cleaning_fee)) %>%
  dplyr::select(cleaning)

deposit = airbnb %>%
  mutate(deposit = ifelse(is.na(security_deposit), 0, security_deposit)) %>%
  dplyr::select(deposit)

# Lets normalize the values to compare them
zdeposit = scale(deposit)
zcleaning = scale(cleaning)
zreviews = scale(airbnb$reviews_per_month)

# plotting reviews vs cleaning
zclean = ggplot(data=airbnb) + 
  geom_density(aes(x=zreviews, colour=I("blue"))) +
  geom_density(aes(x=zcleaning, colour=I("red"))) +
  theme_minimal() +
  labs(x=" Z-scores of reviews and cleaning fees", title="cleaning vs reviews")

# plotting reviews vs deposit
zdep = ggplot(data=airbnb) + 
  geom_density(aes(x=zreviews, colour=I("blue"))) +
  geom_density(aes(x=zdeposit, colour=I("red"))) +
  theme_minimal() +
  labs(x=" Z-scores of reviews and deposit", title="deposit vs reviews")

grid.arrange(zclean, zdep, nrow=1)

# point plot
qplot(reviews_per_month, cleaning, data=airbnb, geom="point", colour = room_type, alpha = 0.2) +
  scale_y_continuous(name="cleaning fees", limits=c(0,200)) +
  theme_minimal() +
  labs(title="cleaning fees / amount of reviews")

qplot(reviews_per_month, deposit, data=airbnb, geom="point") +
  scale_y_continuous(name="deposit amount", limits=c(0,500)) +
  theme_minimal() + 
  labs(title="deposit amount / amount of reviews")

```

From what we can see in the plots, it looks like the higher the cleaning fees and the deposit, the less reviews per month. As the dataset used mentioned that reviews counts for circa 70% of bookings, it may be so that customers refrain from booking places where additional fees are high.


Finally, we run a linear regression to see if we can predict price, and which variables we can use
```{r - Linear regression on Price}

# create dataset with variables we want to check:
airbnb_lm = airbnb_sh %>%
  dplyr::select(price, host_is_superhost, host_listings_count, neighbourhood, room_type, bedrooms, review_scores_rating, number_of_reviews, security_deposit, cleaning_fee) 

# removing missing values
airbnb_lm = airbnb_lm[complete.cases(airbnb_lm), ]

# First, we run a stepwise regression to select variables
null_state = lm(data=airbnb_lm, price ~ 1)
full_state = lm(data=airbnb_lm, price ~ .)

stepwise = stepAIC(null_state, scope=list(lower=null_state, upper=full_state),
                   direction="forward", na.rm=TRUE)

stepwise$anova
summary(stepwise)

model1 = lm(formula = price ~ cleaning_fee + bedrooms + neighbourhood + 
    room_type + security_deposit + review_scores_rating + number_of_reviews + 
    host_is_superhost + host_listings_count, data = airbnb_lm)

summary(model1)

vif(model1) 
# no multicollinearity 
durbinWatsonTest(model1) 
# stat close 2, p-vale > alpha, thus our errors are not autocorrelated

# Cross validation
xvalid = createDataPartition(airbnb_lm$price, p=0.8)
train = airbnb_lm[xvalid$Resample1, ]
test = airbnb_lm[-xvalid$Resample1, ]
str(train)
str(test)

tm = trainControl(method="cv", number=5, returnData =TRUE, returnResamp = "all")

m12 = train(data= train, price ~ cleaning_fee + bedrooms + neighbourhood + 
    room_type + security_deposit + review_scores_rating + number_of_reviews + 
    host_is_superhost + host_listings_count, method ="lm", trControl=tm)

m12
summary(m12)
m12$finalModel
m12$resample

```

Our R^2 is 0.59 and p-value below alpha, thus our model explains 59% of the variance in price for Madrid Airbnb rentals. We deem this to be a model of adequate quality.

In summary, we have looked at the relationships between price and numerous explanatory variables such as neighbourhoods, host_listings_count, superhost status using various statistical techniques. We also performed a linear regression model to see which variables have an impact on price. We hope that this will be useful for potential lister and renters. 
