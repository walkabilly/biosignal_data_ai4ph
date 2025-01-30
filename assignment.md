---
title: "Assignment"
output:
      html_document:
        keep_md: true
---



## Data

The working data for this course is from [https://physionet.org/](https://physionet.org/). We will be using open data provided by Marta Karas,  Jacek Urbanek, Ciprian Crainiceanu, Jaroslaw Harezlak, and William Fadel. We will not using the full dataset and I'm providing and modified version for teaching purposes, which is consistent with the [license](https://physionet.org/content/accelerometry-walk-climb-drive/view-license/1.0.0/) for these data. 

* Karas, M., Urbanek, J., Crainiceanu, C., Harezlak, J., & Fadel, W. (2021). Labeled raw accelerometry data captured during walking, stair climbing and driving (version 1.0.0). PhysioNet. [https://doi.org/10.13026/51h0-a262](https://doi.org/10.13026/51h0-a262).
* Goldberger, A., Amaral, L., Glass, L., Hausdorff, J., Ivanov, P. C., Mark, R., ... & Stanley, H. E. (2000). PhysioBank, PhysioToolkit, and PhysioNet: Components of a new research resource for complex physiologic signals. Circulation. 101 (23), pp. e215–e220. [https://doi.org/10.1161/01.cir.101.23.e215](https://doi.org/10.1161/01.cir.101.23.e215)

## Data Description 

### Study participants

There were 32 healthy participants in the study - 13 men and 19 women - who were of ages ranging between 23 and 52 years. There were 31 right-handed participants; one individual identified themselves as ambidextrous. Participants wore four 3-axial ActiGraph GT3X+ wearable accelerometer devices, placed at left ankle, right ankle, left hip, and left wrist, respectively. ActiLife software was used to synchronize the devices to the same external clock. 

### Data Description 

This project includes raw accelerometry data files, a data files dictionary, and participant demographic information. All data are anonymized. I've cleaned the data to include a subset of 10 participants with demographic data to accelerate the analsysis and teaching for this course. Specifically, the file includes 29,376,900 observations with 10 variables. Variable descriptions are below

* __id__: Unique ID for each participant. <chr>
* __gender__: Self-reported gender of participants. <fct>
* __age__: Age of participants in years. <dbl>
* __weight_lbs__ : Weight of participants in lbs. <dbl>
* __right_handed__ : Right handed yes = 1, no = 0. <fct>
* __activity__: Type of activity (1=walking; 2=descending stairs; 3=ascending stairs; 4=driving; 77=clapping; 99=non-study activity)
* __time_s__: Time from device initiation (seconds [s])
* __lw_x__: Left wrist x-axis measurement (gravitation acceleration [g])
* __lw_y__: Left wrist y-axis measurement (gravitation acceleration [g])
* __lw_z__: Left wrist z-axis measurement (gravitation acceleration [g])

## Data analysis (Part 1)

Your task for data analysis part 1 is the following

1. Read in the data and examine the data using summary statistics.
2. Conduct feature engineering to create 4 new features and describe why you think would would be relevant for classifying the labels in the __activities__ variables
    * Hint. You could calculate the euclidean norm or peak-to-peak amplitude for example.
    * Hint. Provide a written justification for why you think the features are important. 
3. Examine whether your new features by gender or weight status to see if there is potential interaction between these features and the two effect modification variables of interest 
    * Hint. Create a visualization or calculate the summary statistics or your new variables by gender and weight.

#### Assignment submission

Submit your assignments to Canvas via the assignments tab. 

* Submit a .Rmd (R Markdown file) or a knit .html or .pdf file with all your comments and code. 

## Data analysis (Part 2)

Your task for data analysis part 2 is the following

1. Use the features you created in part 1, and develop any new additional features you think might be useful in developing a model to classify the __activities__. 
2. Develop a model that will predict activity class. This can be simplified into a logistic regression problem of `walking/not walking` or something similar, or could be more complex model with multiple classes.  
    * You can decide how far you want to take this. More experience R and ML users might want to go further, but I'm leaving up to each individual. 
3. Examine whether your model performance is similar for men/women or by weight status. 

#### Assignment submission

Submit your assignments to Canvas via the assignments tab. 

* Submit a .Rmd (R Markdown file) or a knit .html or .pdf file with all your comments and code. 


