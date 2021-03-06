---
title       : NYC Real Estate Market Price Estimator
subtitle : estimating house prices based on historical data
author      : Ning Zhang
framework   : io2012        # {io2012, html5slides, shower, dzslides, ...}
highlighter : highlight.js  # {highlight.js, prettify, highlight}
hitheme     : solarized_light      # 
widgets     : []            # {mathjax, quiz, bootstrap}
mode        : selfcontained # {standalone, draft}
knit        : slidify::knit2slides

--- .smaller

## Introduction
  The web-based real estate market price estimator evaluates the market value of a property of interest in the range of New York City. The algorithmic methods behind the estimator are linear regression model with no regularization and KNN model using Euclidean distance metric, both of which are trained with NYC real estate transaction records from May 2013 to December 2015, which are released by NYC Finance.  
  
  The estimator first shows a general view of a portion of sold properties on an NYC map; after the user loads in the information about a property of interest, the estimator presents a prediction of its market value along with a 95% confidence interval, and picks out among sold properties the top 5 nearest neighbors to it; at last, the app calculates geographical coordinates of the five nearest neighbors and the property of interest, illustrating them on map. Users have the discretion in choosing a specific K number of nearest neighbors they want the app to present.  
  
  Although the calculation of geographical coordinates of properties delays the app’s response to users’ requests, the app will get faster when more users load information into the app for estimation. Because the app, upon calculation, updates coordinates of nearest neighbors back into the database, which are remotely stored at Dropbox, it does not need to repeat the calculation when a sold property is picked out for the second time as nearest neighbor.


--- .class #id 

### linear Model Regression

```r
#load in all NYC real estate transaction records from May 2013 to December 2015
allSales <- read.csv("alldataforshinyapp.csv",header = T, stringsAsFactors = FALSE)
lm_prediction <- function(borough, zipcode, land, gross){
    #features of the linear regression model
    features <- c("zip_code", "log_land", "log_gross_sq")
    #response of the model
    output <- 'log_price'
    lm_data <- data_filter(borough)
    if (class(zipcode) == "NULL"){
        zipcode = as.character(unique(lm_data$zip_code)[1])
    }
    lm_data <- lm_data[,c(features,output)]
    lm_data$zip_code <- as.character(lm_data$zip_code)
    test_data <- lm_data[,features][1,]
    test_data$zip_code <- as.character(zipcode)
    test_data$log_land <-  log(land)
    test_data$log_gross_sq <- log(gross)^2
    #list(str(test_data), str(new_data))
    lmFit <- lm(log_price ~ ., data = lm_data)
    predictions <- exp(predict(lmFit, test_data, type = "response", interval = "prediction"))
    predictions <- as.data.frame(predictions)
    names(predictions) = c("Estimated Price in US Dollars","Lower Bound", "Upper Bound")
    predictions
}
```

--- .smaller

### K Nearest Neighbors 

```r
library(FNN)
features_knn <- c("RESIDENTIAL.UNITS", "COMMERCIAL.UNITS", "zip_code", "log_land",
                  "log_gross",  "log_gross_sq", "log_land_sq")
#fit the k nearest neighbors model
knn_prediction <- function(borough, input_res, input_com, input_zipcode,input_land, input_gross, input_k){
    #create customized dataset
    newdata = data_filter(borough)
    #check the class of zip code value
    if (class(input_zipcode) == "NULL"){input_zipcode = unique(newdata$zip_code)[1]}
    #prepare test data and labels
    labels <- newdata$log_price
    knn_data = newdata[,features_knn]
    knn_data$zip_code <- as.numeric(knn_data$zip_code)
    test_knn <- knn_data[1,]
    test_knn$RESIDENTIAL.UNITS = input_res; test_knn$COMMERCIAL.UNITS = input_com
    test_knn$zip_code = as.numeric(input_zipcode); test_knn$log_land = log(input_land)
    test_knn$log_gross = log(input_gross); test_knn$log_gross_sq = test_knn$log_land^2
    test_knn$log_gross_sq = test_knn$log_gross^2
    #fit the knn model
    kFit <- knn(knn_data,test_knn[1,], labels, k = input_k, algorithm = "brute")
    indices = attr(kFit, "nn.index")
    newdata[indices,]}

show_features <- c("BOROUGH","NEIGHBORHOOD","ADDRESS", "ZIP.CODE", "RESIDENTIAL.UNITS", "COMMERCIAL.UNITS",
                   "LAND.SQUARE.FEET", "GROSS.SQUARE.FEET", "YEAR.BUILT","SALE.PRICE", "SALE.DATE")
```

---

### Estimation Output

```r
#create new subset of data according to the borough users choosed
data_filter <- function(borough){allSales[allSales$BOROUGH == as.character(borough),]}
#house input information
borough = "BROOKLYN";zipcode = "11201";land_area_SquareFeet = 1000
gross_lot_area_SquareFeet = 1000;commercial_units = 0;residential_units = 1
#linear regression model predicts the estimated value of the house and its 95% CI
prediction <- lm_prediction(borough, zipcode, land_area_SquareFeet, gross_lot_area_SquareFeet)
#linear model output
prediction
```

```
##   Estimated Price in US Dollars Lower Bound Upper Bound
## 1                       1464885    510358.5     4204668
```

```r
#KNN model predicts the nearest neighbor of the hosue when K=1.
knn_output <- knn_prediction(borough, residential_units, commercial_units,
                             zipcode,land_area_SquareFeet, gross_lot_area_SquareFeet, input_k=1)
#KNN output
knn_output[,show_features]
```

```
##       BOROUGH NEIGHBORHOOD             ADDRESS ZIP.CODE RESIDENTIAL.UNITS
## 9539 BROOKLYN  BOERUM HILL  269 PACIFIC STREET    11201                 0
##      COMMERCIAL.UNITS LAND.SQUARE.FEET GROSS.SQUARE.FEET YEAR.BUILT
## 9539                1             1527              1025       2004
##      SALE.PRICE  SALE.DATE
## 9539 $2,000,000 2014-02-14
```
Finaly, all geo coordinates of the property of interest and its nearest neighbors are calculated with ggmap package, and illustrated on map with leaflet package.
