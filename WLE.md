---
title: "WeightLiftingExercise"
author: "SE_CPT"
date: "7th January 2021"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Introduction

As part of the Data Science Specialisation run by JHU on Coursera the Practical Machine Learning module asks trainees to analyse some data on weight lifting exercises - specifically "unilateral dumbbell biceps curl".  The data includes the correct manner of completing the exercise "A" and four common mistakes "B, C, D & E".  

The task is to predict how well people carry out the weight lifting exercise and categorise which method was used.

>**Data Citation:**
>Velloso, E.; Bulling, A.; Gellersen, H.; Ugulino, W.; Fuks, H. Qualitative Activity Recognition of Weight Lifting >Exercises. Proceedings of 4th International Conference in Cooperation with SIGCHI (Augmented Human '13) . >Stuttgart, Germany: ACM SIGCHI, 2013.
>
>Read more: http://groupware.les.inf.puc-rio.br/har

## Load Libraries

Load the libraries necessary to run the code.

```{r Load libraries, results = 'hold'}
## Load relevant libraries
library(caret)
library(ggplot2)
library(viridisLite)
```

## Import data

Reading the data from the source and importing it into training and testing datasets.

```{r data, cache = TRUE, results = 'hold'}
## Download data
myurl1 <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv"
myurl2 <- "https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv"
download.file(myurl1,"file1.csv")
download.file(myurl2,"file2.csv")
## Read the raw data
traindata <- read.csv("file1.csv")
testdata <- read.csv("file2.csv") # for test purposes only - don't touch until end
```

## Partition Data

To enable testing of the model against a subset of the original training data a partition was created.  
This will allow cross-validation to be carried out and information about the accuracy of the model to be gathered.

```{r Partition Data, results = 'hold'}
## Partition Data for testing and training
set.seed(32343)
inTrain <- createDataPartition(y = traindata$classe, 
                               p = 0.75, 
                               list = FALSE) ## 75% for training
training <- traindata[inTrain,]
testing <- traindata[-inTrain,]
```

## Explore Data

Exploration of the data is important to allow decisions about cleaning data and how to model it later.  I will only explore the subset of the data I have set aside for training.

```{r Explore, results = 'hold'}
## Explore data to allow decisions to be made
dim(training)
str(training)
```

## Clean Data

Following the exploration it was clear some variables had NA values.  I decided to remove columns with NAs in if the total number of NAs in the column exceeded 95% of the length of the column as it wouldn't be possible to substitute data in this case.  Following this I did an assessment to remove any variables that had nearly no effect.  This helped reduce the size of the dataset before creating a model.

```{r Clean Data, results = 'hold'}
## Check and remove NAs if more than 95% of total column
cNames <- names(training)  # find column names
na <- 0
ratio <- 0
kill <- 0
tot <- length(training$max_roll_belt)
for (n in 1:(length(cNames)-1)) 
{
  na[n] <- sum(is.na(training[,n]))
  ratio[n] <- na[n]/tot

  if (ratio[n] > 0.95) {
    kill[n] <- n
  } else {
    kill[n] <- NA
  }
}
killoff <- na.omit(kill)
cleanData <- training[,-killoff]

## Remove variables with nearly no effect from training set
nsv <- nearZeroVar(cleanData, 
                   saveMetrics = FALSE)
cutTraining <- cleanData[,-nsv]
```

## Modeling

The method used to train the model to predict the classes was "gbm" Stochastic Gradient Boosting.  The method was applied to the clean training dataset.  The first 6 columns of the data were excluded from the prediction factors as I didn't want to base the model on the names of the participants or the time at which the exercises were done.  No further pre-processing was carried out.

Then cross-validation was carried out against the subset of testing data that is 25% of the original training data provided.  A confusion matrix is created and data gathered to enable the confusion matrices to be plotted later.

```{r Modeling, cache = TRUE, results = 'hold'}
## Train model
modelFit <- train(classe ~ ., 
                  method = "gbm", 
                  data = cutTraining[-c(1:6)],
                  verbose = FALSE)

## Cross Validation
predictionGMB <- predict(modelFit, 
                 testing)
## Out of sample
testing$predRight <- predictionGMB==testing$classe
table(testing$predRight)

## Plotting data
data <- data.frame(table(predictionGMB, 
                         testing$classe))
dataPerCent <- data.frame(prop.table(table(predictionGMB, 
                                           testing$classe), 2))
## Confusion matrix
cm <- confusionMatrix(predictionGMB, 
                      testing$classe)
cm
```

## Plots

Confusion matrices were plotted to show the frequency that the model predicted accurately.

```{r plots, results = 'hold'}
## Plot initial Confusion Matrix
plot1 <- ggplot(data = data,
                mapping = aes(x = predictionGMB,
                              y = Var2)) 
plot1 <- plot1 + geom_tile(mapping = aes(fill = Freq)) 
plot1 <- plot1 + geom_text(mapping = aes(label = sprintf("%1.0f", Freq)),
                           vjust = 1) 
plot1 <- plot1 + labs(x = "Prediction", 
                      y = "Actual", 
                      title = "Confusion Matrix")
plot1 <- plot1 + scale_fill_distiller(name = "Frequency")
plot1

## Plot Normalised Confusion Matrix
plot2 <- ggplot(data = dataPerCent,
                mapping = aes(x = predictionGMB,
                              y = Var2)) 
plot2 <- plot2 + geom_tile(mapping = aes(fill = Freq)) 
plot2 <- plot2 + geom_text(mapping = aes(label = sprintf("%1.2f", Freq)),
                           vjust = 1) 
plot2 <- plot2 + labs(x = "Prediction", 
                      y = "Actual", 
                      title = "Normalised Confusion Matrix")
plot2 <- plot2 + scale_fill_distiller(name = "Normalised Frequency")
plot2
```

## Discussion of Results

The corss-validation exercise demostrates that the model predicts which class of the weight lifting exercise was actually carried out with a high degree of accuracy - an overall accuracy of `r cm$overall[[1]]`.  There are very few false positives and false negatives - therefore the out of sample error is low (`r 1-cm$overall[[1]]`).

Finally the model is tested against the original test data set of 20 data points as the final test.  This is outputted as a dataframe to show which prediction answers which question for the quiz.

```{r Test, results = 'hold'}
predTest <- predict(modelFit, testdata)
predTest <- data.frame(predTest)
predTest
```