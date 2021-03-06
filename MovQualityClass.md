# Movement Quality Classification
Denis Willett  
July 24, 2014  

### Executive Summary
> The increasing popularity of wearable computing devices for monitoring activity has stimulated interest in measuring both the quantity and, recently, the quality of movement.  To explore whether the quality of movement patterns can be classified, data from students performing supervised bicep curls correctly and incorrectly were used for model training.  The final random forest model had a within sample accuracy of 100% and an estimated out of sample accuracy of 99% suggesting that machine learning techniques can be used to classify movement quality.  

### Author

This report was generated by Denis Willett on July 24, 2012 in partial fulfillment of the requirements for the Practical Machine Learning Coursera Course taught by faculty in Biostatistics from the Johns Hopkins Bloomberg School of Public Health.

### Data

The dataset was formulated to examine exercise quality in test subjects.  To that end,  subjects performed 10 repititions of unilateral dumbbell bicep curls correctly (Class A) and incorrectly under supervision while wearing forearm, arm, waist, and dumbbell sensors.   Incorrect exercises were  performed in four supervised specific ways (Classes B, C, D, and E).  Data are provided courtesy of the Human Activity Recognition project at the Pontifical Catholic University of Rio de Janeiro; more information on study design can be found [here](http://groupware.les.inf.puc-rio.br/har).  The complete dataset was then split into [training](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv) and [test](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv) sets for use in the course project for the [Practical Machine Learning Coursera Class](https://www.coursera.org/course/predmachlearn) as part of the [Data Science Specialization](https://www.coursera.org/specialization/jhudatascience/1?utm_medium=listingPage) taught in collaboration with the Johns Hopkins Bloomberg School of Public Health.  



Data Processing
--------------------------------------------

### Environment Setup



Global options were set to facilitate readability and reproducibility.  

```r
opts_chunk$set(fig.width=8, fig.height=6,
               echo=TRUE, warning=FALSE, message=FALSE,
               tidy=TRUE)
```

A variety of R packages were used to facilitate processsing and analysis:

* *__caret__* was used for data processing and model training.  
* *__doParallel__* was used for parallelization of model fitting.  
* *__dplyr__* was used for fast filtering and sorting of the data.   
* *__ggplot2__* was used for plotting.  
* *__knitr__* was used for document formatting.  
* *__pander__* was used for table formatting.  
* *__randomForest__* was used for model training.  

Citations for the packages used can be found in the __Works Cited__ section at the end of the document.  


```r
library(caret)
library(doParallel)
library(dplyr)
library(ggplot2)
library(knitr)
library(pander)
library(randomForest)
library(reshape2)
```

Various other functions were used to facilitate analysis and plotting:

Two functions were provided by [Sebastian Kranz](https://gist.github.com/skranz) as helper functions for _dplyr_ that allow selection of columns with character string input.  


```r
eval.string.dplyr = function(.data, .fun.name, ...) {
    args = list(...)
    args = unlist(args)
    code = paste0(.fun.name, "(.data,", paste0(args, collapse = ","), ")")
    df = eval(parse(text = code, srcfile = NULL))
    df
}

s_select = function(.data, ...) {
    eval.string.dplyr(.data, "select", ...)
}
```


### Data Loading


```r
train <- read.csv("pml-training.csv")
test <- read.csv("pml-testing.csv")
```

### Data Cleaning

Many columns of the data were rife with NA values, divide by zero errors, and blanks.  Errors and blanks were replaced with NA.  


```r
train[which(train == "", arr.ind = TRUE)] <- NA
train[which(train == "#DIV/0!", arr.ind = TRUE)] <- NA
```

Columns with more than 80% NAs were set aside for discard.  


```r
cutoff <- nrow(train) * 0.8

na.values <- train %>% select(classe, contains("belt"), contains("arm"), contains("bell")) %>% 
    summarise_each(funs(sum(is.na(.)) > cutoff))

no.na <- colnames(na.values)[which(na.values == FALSE, arr.ind = TRUE)[, 2]]
```

Many features reported variables with very low variance or many non-unique variables.  Due to their homogeneity, these features would be poor predictors of exercise performance.  Those features with less than 80% NAs and greater than 5% unique values were selected for consideration in the model.  


```r
keptcols <- train %>% s_select(no.na) %>% select(-classe) %>% nearZeroVar(., 
    saveMetrics = TRUE) %>% mutate(rowname = no.na[-1]) %>% filter(percentUnique >= 
    5)

minTrain <- train %>% s_select(keptcols$rowname) %>% cbind(., classe = train$classe)
```

Model Development
------------------------------------------------------------------

### Preprocessing

The pared down training data were split (with a set seed for reproducibility) for eventual estimation of out of sample error.  Because the test data set supplied in the course did not contain classifications, the true out of sample error cannot be calculated.  For the remainder of the __Model Development__ section, training and test sets refer to subsets of the training dataset supplied in the course, not the full datasets themselves.  


```r
set.seed(24)
inTrain <- createDataPartition(minTrain$classe, p = 0.75, list = FALSE)

mintraining <- minTrain[inTrain, ]
mintesting <- minTrain[-inTrain, ]
```

Ten fold cross validation was used to minimize out of sample prediction error.  Seeds were set for reproducibility.  


```r
set.seed(24)
seeds = vector(mode = "list", length = 11)
for (i in 1:10) {
    seeds[[i]] <- sample.int(1000, 3)
}

seeds[[11]] <- sample.int(1000, 1)

fitControl <- trainControl(method = "cv", number = 10, seeds = seeds)
```



### Training

A random forest model was used for classification.  Due to the computationally intense nature of the algorithm, the model was trained in parallel.  Parallelization reduced training time to less than 6 minutes.  


```r
registerDoParallel(cores = detectCores())
partime <- system.time(minmodpar <- train(classe ~ ., method = "rf", data = mintraining, 
    trControl = fitControl))
```

### Model Results






The within sample accuracy (calculated on the training set) for the random forest model developed above was 100% with the lower bound of the 95% confidence interval at 100%.  


--------------------------------
&nbsp;   A    B    C    D    E  
------- ---- ---- ---- ---- ----
 **A**  4185  0    0    0    0  

 **B**   0   2848  0    0    0  

 **C**   0    0   2567  0    0  

 **D**   0    0    0   2412  0  

 **E**   0    0    0    0   2706
--------------------------------

Table: Table 1: Within Sample Predictions (left column) vs References (column names)


The out of sample accuracy (calculated on the test set split from the original full training set) was estimated at 99.2 with the 95% confidence interval ranging from 98.9% to 99.4%.  


----------------------------
&nbsp;   A    B   C   D   E 
------- ---- --- --- --- ---
 **A**  1390  6   0   0   0 

 **B**   3   939  7   0   2 

 **C**   2    4  845  5   2 

 **D**   0    0   3  797  3 

 **E**   0    0   0   2  894
----------------------------

Table: Table 2: Out of Sample Predictions (left column) vs References (column names)


```r
insamp.pre <- predict(minmodpar, mintraining)
insamp <- confusionMatrix(insamp.pre, mintraining$classe)
outsamp.pre <- predict(minmodpar, mintesting)
outsamp <- confusionMatrix(outsamp.pre, mintesting$classe)

pander(insamp$table, caption = "Table 1: Within Sample Predictions vs References")
pander(outsamp$table, caption = "Table 2: Out of Sample Predictions vs References")
```

### Model Diagnostics


```r
varImportance <- data.frame(vars = rownames(importance(minmodpar$final)), importance(minmodpar$final))

varImportance$vars <- reorder(varImportance$vars, varImportance$MeanDecreaseGini)

ggplot(varImportance, aes(x = vars, y = MeanDecreaseGini)) + geom_point() + 
    theme_bw(14) + labs(x = "", y = "Mean Decrease in Gini Score") + coord_flip()
```

> __Figure 1:__ Variable Importance.  Mean decrease in Gini scores for variables included in the final random forest model.  Note that metrics associated with movement of the belt sensor are most important, followed by pitch and roll of the forearm.  This may make sense intuitively: movement of the belt during dumbbell curls is indicative of poor quality movement (especially for Class E - throwing the hips to the front) while pitch and roll of the forearm are indicative of poor dumbbell stability.  
>
![plot of chunk Figure_One](./MovQualityClass_files/figure-html/Figure_One.png) 



```r
err.info <- data.frame(minmodpar$final$err.rate, ntrees = 1:500)
m.err.info <- melt(err.info, id = "ntrees", variable.name = "Type", value.name = "Error")

ggplot(m.err.info, aes(x = ntrees, y = Error, linetype = Type)) + geom_line() + 
    theme_bw(14) + labs(x = "Number of Trees")
```


> __Figure 2:__ Out of bag (OOB) and class (A,B,C,D,E) error estimates from the final random forest model.  
>
![plot of chunk Figure_Two](./MovQualityClass_files/figure-html/Figure_Two.png) 

## Prediction

The model trained above was then applied to the 20 observations in the test set which generated the following predictions:


```r
pander(cbind(Problem_ID = test$problem_id, Prediction = as.character(predict(minmodpar, 
    test))))
```


-------------------------
 Problem_ID   Prediction 
------------ ------------
     1            B      

     2            A      

     3            B      

     4            A      

     5            A      

     6            E      

     7            D      

     8            B      

     9            A      

     10           A      

     11           B      

     12           C      

     13           B      

     14           A      

     15           E      

     16           E      

     17           A      

     18           B      

     19           B      

     20           B      
-------------------------

## Conclusion 

The high degree of within sample and estimated out of sample accuracy suggest that machine learning techniques, given appropriate training data, can be used to successfully classify quality of movement.  Given the widespread adoption of wearable computing devices, the impact of movement quality classification can be large.  Applications of these methods could include monitoring and training athletic performance,  monitoring stability and safety of senior citizens, and improving injury rehabilitation protocols.  


## Works Cited

__Daróczi, G. (2013)__. pander: An R Pandoc Writer. R package
  version 0.3.8, URL http://cran.r-project.org/package=pander

__Max Kuhn__. Contributions from Jed Wing, Steve Weston, Andre
  Williams, Chris Keefer, Allan Engelhardt, Tony Cooper, Zachary
  Mayer and the R Core Team (2014). caret: Classification and
  Regression Training. R package version 6.0-30.
  http://CRAN.R-project.org/package=caret
  
__A. Liaw and M. Wiener (2002)__. Classification and Regression by
  randomForest. R News 2(3), 18--22.
  
__Revolution Analytics and Steve Weston (2014)__. doParallel:
  Foreach parallel adaptor for the parallel package. R package
  version 1.0.8. http://CRAN.R-project.org/package=doParallel
  
__Hadley Wickham__. ggplot2: elegant graphics for data analysis.
  Springer New York, 2009.

__Hadley Wickham (2007)__. Reshaping Data with the reshape Package.
  Journal of Statistical Software, 21(12), 1-20. URL
  http://www.jstatsoft.org/v21/i12/.
  
__Hadley Wickham and Romain Francois (2014)__. dplyr: dplyr: a
  grammar of data manipulation. R package version 0.2.
  http://CRAN.R-project.org/package=dplyr

__Yihui Xie (2014)__. knitr: A general-purpose package for dynamic
  report generation in R. R package version 1.6.

## Session Info


```r
sessionInfo()
```

```
## R version 3.1.1 (2014-07-10)
## Platform: x86_64-apple-darwin10.8.0 (64-bit)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] parallel  stats     graphics  grDevices utils     datasets  methods  
## [8] base     
## 
## other attached packages:
##  [1] e1071_1.6-3        reshape2_1.4       randomForest_4.6-7
##  [4] pander_0.3.8       dplyr_0.2          doParallel_1.0.8  
##  [7] iterators_1.0.7    foreach_1.4.2      caret_6.0-30      
## [10] ggplot2_1.0.0      lattice_0.20-29    knitr_1.6         
## 
## loaded via a namespace (and not attached):
##  [1] assertthat_0.1      BradleyTerry2_1.0-5 brglm_0.5-9        
##  [4] car_2.0-20          class_7.3-10        codetools_0.2-8    
##  [7] colorspace_1.2-4    digest_0.6.4        evaluate_0.5.5     
## [10] formatR_0.10        grid_3.1.1          gtable_0.1.2       
## [13] gtools_3.4.1        htmltools_0.2.4     labeling_0.2       
## [16] lme4_1.1-6          magrittr_1.0.1      MASS_7.3-33        
## [19] Matrix_1.1-4        minqa_1.2.3         munsell_0.4.2      
## [22] nlme_3.1-117        nnet_7.3-8          plyr_1.8.1         
## [25] proto_0.3-10        Rcpp_0.11.2         RcppEigen_0.3.2.1.2
## [28] rmarkdown_0.2.49    scales_0.2.4        splines_3.1.1      
## [31] stringr_0.6.2       tools_3.1.1         yaml_2.1.13
```















