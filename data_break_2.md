---
title: "Data Break 2"
output:
      html_document:
        keep_md: true
---



### Read in data


``` r
data <- read_csv("data_clean.csv")
```

```
## Rows: 2937600 Columns: 10
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (2): id, gender
## dbl (8): age, weight_lbs, right_handed, activity, time_s, lw_x, lw_y, lw_z
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

``` r
glimpse(data)
```

```
## Rows: 2,937,600
## Columns: 10
## $ id           <chr> "id1c7e64ad", "id1c7e64ad", "id1c7e64ad", "id1c7e64ad", "…
## $ gender       <chr> "female", "female", "female", "female", "female", "female…
## $ age          <dbl> 54, 54, 54, 54, 54, 54, 54, 54, 54, 54, 54, 54, 54, 54, 5…
## $ weight_lbs   <dbl> 165, 165, 165, 165, 165, 165, 165, 165, 165, 165, 165, 16…
## $ right_handed <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
## $ activity     <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
## $ time_s       <dbl> 463.08, 445.32, 445.28, 463.07, 463.15, 445.27, 463.06, 4…
## $ lw_x         <dbl> -0.090, 0.066, 0.012, -0.082, -0.063, -0.004, -0.070, -0.…
## $ lw_y         <dbl> -0.684, -1.133, -0.820, -0.695, -0.664, -0.770, -0.715, -…
## $ lw_z         <dbl> 0.016, 0.090, 0.031, 0.020, 0.016, 0.008, 0.027, 0.035, -…
```

``` r
### Sort by id and time
data <- arrange(data, id, time_s)
data$id <- as.factor(data$id)

## Filter out non study activity
data <- dplyr::filter(data, activity != 99)

### Converting appropriate variables to factor
data <- data %>% mutate_at(c(2, 5, 6), factor)

### Creating a second level variable

data$time <- gsub("\\..*","", data$time_s)
```

### Feature development


``` r
### Vector Magnitude
data$vec_mag <- sqrt((data$lw_x^2) + (data$lw_y^2) + (data$lw_z^2))

### 5 second Moving average of x, y, z

data$roll_mean_x <- rollmean(data$lw_x, 500, fill = mean(data$lw_x))
data$roll_mean_y <- rollmean(data$lw_y, 500, fill = mean(data$lw_y))
data$roll_mean_z <- rollmean(data$lw_z, 500, fill = mean(data$lw_z))

### Correlations between x, y, z

data_xyz <- select(data, lw_x, lw_y, lw_z)

### Correlation x y 
cor(data$lw_x, data$lw_y)
```

```
## [1] -0.1848453
```

``` r
data$corr_x_y <- rollapply(data_xyz, width=500, function(x) cor(x[, 1], x[, 2]), by.column=FALSE, fill = -0.1848453)

### Correlation y z 
cor(data$lw_y, data$lw_z)
```

```
## [1] -0.4676048
```

``` r
data$corr_y_z <- rollapply(data_xyz, width=500, function(x) cor(x[, 2], x[, 3]), by.column=FALSE, fill = -0.2885716)

### Correlation x z 
cor(data$lw_x, data$lw_z)
```

```
## [1] 0.0996406
```

``` r
data$corr_x_z <- rollapply(data_xyz, width=500, function(x) cor(x[, 1], x[, 3]), by.column=FALSE, fill = 0.1069972)

#write_csv(data, "data_rf.csv")

rm(data_xyz)
#rm(data)
```

### Random Forest 

We will use the Tidymodels framework for model fitting 

#### Data split with 70/30 (Do not recommend)


``` r
#data <- read_csv("data_rf.csv")
#data$activity <- as.factor(data$activity)

# Fix the random numbers by setting the seed 
# This enables the analysis to be reproducible when random numbers are used 
set.seed(10)

data <- select(data, id, time_s, gender, age, weight_lbs, activity, lw_x, lw_y, lw_z, vec_mag, roll_mean_x, roll_mean_y, roll_mean_z, corr_x_y, corr_y_z, corr_x_z)

#### Cross Validation Split
cv_split <- initial_validation_split(data, 
                            strata = activity, 
                            prop = c(0.7, 0.2))

# Create data frames for the two sets:
train_data <- training(cv_split)
table(train_data$activity)
```

```
## 
##      1      2      3      4     77 
## 351882  56598  61490 871829  27340
```

``` r
test_data  <- testing(cv_split)
table(test_data$activity)
```

```
## 
##      1      2      3      4     77 
##  50256   8138   8976 124308   3914
```

High risk of data leakage with this method. We can do k-fold cross validation to make this less likely and improve the robustness of our models. 

### Model 

Here we use the tidy models to setup a model using `ranger` and `classification` and we call the specific model we want to fit. I'm fixing mtry = 5, min_n = 10, and tress = 10 just to make the model run more efficiently in class. Normally you want to tune these to find the optimal values. This optimization process can take a long time and is best done using parallel processing. 


``` r
cores <- parallel::detectCores()
cores
```

```
## [1] 8
```


``` r
rf_model <- rand_forest(mtry = 5, min_n = 10, trees = 10) %>% 
              set_engine("ranger", num.threads = cores) %>% 
              set_mode("classification")
```

#### Recipe

The recipe() function as we used it here has two arguments

1. A formula. Any variable on the left-hand side of the tilde (~) is considered the model outcome (here, arr_delay). On the right-hand side of the tilde are the predictors. Variables may be listed by name, or you can use the dot (.) to indicate all other variables as predictors.
2. The data. A recipe is associated with the data set used to create the model. This will typically be the training set, so data = train_data here. Naming a data set doesn’t actually change the data itself; it is only used to catalog the names of the variables and their types, like factors, integers, dates, etc.

Now we can add roles to this recipe. We can use the update_role() function to let recipes know that `ADM_STUDY_ID` is a variable with a custom role that we called "ID" (a role can have any character value). Whereas our formula included all variables in the training set other than bmi_recode as predictors (that's what the `.` does), this tells the recipe to keep these two variables but not use them as either outcomes or predictors.


``` r
activity_recipe <- 
  recipe(activity ~ ., data = train_data) %>% 
  update_role(id, new_role = "ID") %>% 
  update_role(time_s, new_role = "ID") %>%
  step_zv(all_predictors()) ### Remove columns from the data when the training set data have a single value. Zero variance predictor
```

#### Create a workflow

A workflow connects our recipe with out model. The workflow let's us setup the models without actually have run things over and over again. This is helpful because as you will sometimes models can take a long time to run. 


``` r
activity_workflow <- 
        workflow() %>% 
        add_model(rf_model) %>% 
        add_recipe(activity_recipe)

activity_workflow
```

```
## ══ Workflow ════════════════════════════════════════════════════════════════════
## Preprocessor: Recipe
## Model: rand_forest()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## 1 Recipe Step
## 
## • step_zv()
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Random Forest Model Specification (classification)
## 
## Main Arguments:
##   mtry = 5
##   trees = 10
##   min_n = 10
## 
## Engine-Specific Arguments:
##   num.threads = cores
## 
## Computational engine: ranger
```

### Fit a model 


``` r
set.seed(100)

folds <- vfold_cv(train_data, v = 5) ## normally you would do at least 10 folds. Just doing 5 because it's faster.

activity_fit <- 
      activity_workflow %>% 
      fit_resamples(resamples = folds,
                    control = control_resamples(save_pred = TRUE, 
                                                verbose = FALSE)) ## Edit for running live

activity_fit
```

```
## # Resampling results
## # 5-fold cross-validation 
## # A tibble: 5 × 5
##   splits                   id    .metrics         .notes           .predictions
##   <list>                   <chr> <list>           <list>           <list>      
## 1 <split [1095311/273828]> Fold1 <tibble [3 × 4]> <tibble [0 × 3]> <tibble>    
## 2 <split [1095311/273828]> Fold2 <tibble [3 × 4]> <tibble [0 × 3]> <tibble>    
## 3 <split [1095311/273828]> Fold3 <tibble [3 × 4]> <tibble [0 × 3]> <tibble>    
## 4 <split [1095311/273828]> Fold4 <tibble [3 × 4]> <tibble [0 × 3]> <tibble>    
## 5 <split [1095312/273827]> Fold5 <tibble [3 × 4]> <tibble [0 × 3]> <tibble>
```

### Collect the results


``` r
rf_best <- 
  activity_fit %>% 
  select_best(metric = "roc_auc")

rf_best
```

```
## # A tibble: 1 × 1
##   .config             
##   <chr>               
## 1 Preprocessor1_Model1
```

``` r
rf_auc_fit <- 
  activity_fit %>% 
  collect_predictions(parameters = rf_best) 
```

### Metrics

#### Confusion Matrix

We can generate a confusion matrix by using the `conf_mat()` function by supplying the data frame (`rf_auc_fit`), the truth column `activity` and predicted class `.pred_class` in the estimate attribute.

A confusion matrix is sort of a 2x2 table with the true values on one side and predicted values in another column. If we look on the diagonal we see when the model correctly predicts the values `yes/no` and off diagonal is when the model does not predict the correct value.

Here is the confusion matrix for one fold. 


``` r
cm <- rf_auc_fit %>%
        dplyr::filter(id == "Fold1") %>%
        conf_mat(activity, .pred_class)
cm
```

```
##           Truth
## Prediction      1      2      3      4     77
##         1   70198     57     88      5     28
##         2      21  11139     11      0     45
##         3      18     11  12068      0     28
##         4      31      0      2 174605     41
##         77     21     41     18      2   5350
```

Here is the confusion matrix for all 5 of the folds. 


``` r
conf_mat(rf_auc_fit, truth = activity,
         estimate = .pred_class)
```

```
##           Truth
## Prediction      1      2      3      4     77
##         1  351376    299    380     51    125
##         2     121  56071     62      1    188
##         3     103     48  60902      4    145
##         4     171      3     25 871752    223
##         77    111    177    121     21  26659
```

#### Accuracy

We can calculate the classification accuracy by using the `accuracy()` function by supplying the final data frame `rf_auc_fit`, the truth column `activity` and predicted class `.pred_class` in the estimate attribute. 


``` r
accuracy(rf_auc_fit, truth = activity,
         estimate = .pred_class)
```

```
## # A tibble: 1 × 3
##   .metric  .estimator .estimate
##   <chr>    <chr>          <dbl>
## 1 accuracy multiclass     0.998
```



``` r
sessionInfo()
```

```
## R version 4.4.2 (2024-10-31)
## Platform: aarch64-apple-darwin20
## Running under: macOS Sequoia 15.3.1
## 
## Matrix products: default
## BLAS:   /Library/Frameworks/R.framework/Versions/4.4-arm64/Resources/lib/libRblas.0.dylib 
## LAPACK: /Library/Frameworks/R.framework/Versions/4.4-arm64/Resources/lib/libRlapack.dylib;  LAPACK version 3.12.0
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## time zone: America/Regina
## tzcode source: internal
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
##  [1] ranger_0.17.0         zoo_1.8-12            yardstick_1.3.1      
##  [4] workflowsets_1.1.0    workflows_1.1.4       tune_1.2.1           
##  [7] rsample_1.2.1         recipes_1.1.0         parsnip_1.2.1        
## [10] modeldata_1.4.0       infer_1.0.7           dials_1.3.0          
## [13] scales_1.3.0          broom_1.0.7           tidymodels_1.2.0     
## [16] gsignal_0.3-7         signal_1.8-1          seewave_2.2.3        
## [19] BiostatsUHNplus_1.0.1 lubridate_1.9.3       forcats_1.0.0        
## [22] stringr_1.5.1         dplyr_1.1.4           purrr_1.0.2          
## [25] readr_2.1.5           tidyr_1.3.1           tibble_3.2.1         
## [28] ggplot2_3.5.1         tidyverse_2.0.0      
## 
## loaded via a namespace (and not attached):
##   [1] tensorA_0.36.2.1    rstudioapi_0.17.1   jsonlite_1.8.9     
##   [4] magrittr_2.0.3      TH.data_1.1-2       estimability_1.5.1 
##   [7] nloptr_2.1.1        rmarkdown_2.29      vctrs_0.6.5        
##  [10] minqa_1.2.8         rstatix_0.7.2       htmltools_0.5.8.1  
##  [13] Formula_1.2-5       sass_0.4.9          parallelly_1.39.0  
##  [16] bslib_0.8.0         plyr_1.8.9          sandwich_3.1-1     
##  [19] emmeans_1.10.6      cachem_1.1.0        iterators_1.0.14   
##  [22] lifecycle_1.0.4     pkgconfig_2.0.3     Matrix_1.7-1       
##  [25] R6_2.5.1            fastmap_1.2.0       future_1.34.0      
##  [28] clue_0.3-66         digest_0.6.37       numDeriv_2016.8-1.1
##  [31] colorspace_2.1-1    spatial_7.3-17      furrr_0.3.1        
##  [34] MCMCglmm_2.36       fansi_1.0.6         timechange_0.3.0   
##  [37] abind_1.4-8         compiler_4.4.2      bit64_4.5.2        
##  [40] withr_3.0.2         backports_1.5.0     carData_3.0-5      
##  [43] MASS_7.3-61         lava_1.8.0          corpcor_1.6.10     
##  [46] fBasics_4041.97     tools_4.4.2         ape_5.8-1          
##  [49] zip_2.3.1           future.apply_1.11.3 nnet_7.3-19        
##  [52] glue_1.8.0          stabledist_0.7-2    nlme_3.1-166       
##  [55] grid_4.4.2          cluster_2.1.6       reshape2_1.4.4     
##  [58] generics_0.1.3      gtable_0.3.6        tzdb_0.4.0         
##  [61] class_7.3-22        data.table_1.16.2   hms_1.1.3          
##  [64] car_3.1-3           utf8_1.2.4          rmutil_1.1.10      
##  [67] foreach_1.5.2       pillar_1.9.0        vroom_1.6.5        
##  [70] lhs_1.2.0           splines_4.4.2       lattice_0.22-6     
##  [73] bit_4.5.0           survival_3.7-0      tidyselect_1.2.1   
##  [76] knitr_1.49          xfun_0.49           hardhat_1.4.0      
##  [79] timeDate_4041.110   stringi_1.8.4       DiceDesign_1.10    
##  [82] yaml_2.3.10         boot_1.3-31         evaluate_1.0.1     
##  [85] codetools_0.2-20    timeSeries_4041.111 cli_3.6.3          
##  [88] rpart_4.1.23        xtable_1.8-4        munsell_0.5.1      
##  [91] tuneR_1.4.7         jquerylib_0.1.4     afex_1.4-1         
##  [94] Rcpp_1.0.13-1       globals_0.16.3      stable_1.1.6       
##  [97] coda_0.19-4.1       parallel_4.4.2      modeest_2.4.0      
## [100] ggh4x_0.3.0         gower_1.0.1         cubature_2.1.1     
## [103] GPfit_1.0-8         lme4_1.1-35.5       listenv_0.9.1      
## [106] mvtnorm_1.3-2       ipred_0.9-15        lmerTest_3.1-3     
## [109] prodlim_2024.06.25  crayon_1.5.3        openxlsx_4.2.7.1   
## [112] statip_0.2.3        rlang_1.1.4         cowplot_1.1.3      
## [115] multcomp_1.4-26
```





