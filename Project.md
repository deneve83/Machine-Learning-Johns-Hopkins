---
title: "Practical Machine Learning Project"
author: "DRD"
date: "10/3/2021"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Human Activity Recognition

One thing that people regularly do is quantify how  much of a particular activity they do, but they rarely quantify how well they do it. In this project, our goal is to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants.


Wearable devices allow us to collect a large amount of data about personal activity in a manner much less expensive than prior technology allowed. The learning objective is to create a model to predict how well a bicep curl was performed using the data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. 

Participants performed barbell lifts correctly and incorrectly in 5 different ways.
-In accordance with specification (Class A)
-Throwing the elbows to the front (Class B) - mistake
-Lifting the dumbbell only halfway (Class C) - mistake
-Lowering the dumbbell only halfway (Class D) - mistake
-Throwing the hips to the front (Class E) - mistake


###Data Preparation
The data appears at first to have a lot of NA values. The data is provided with aggregated statistical metrics across each window of observation. Due to size of the training sample (19622 observations and up to 60 variables), parallel processing was selected for model development

```{r Data Prep, cache = TRUE}

set.seed(1603)

#install.packages("doParallel")
#install.packages("randomForest")
#install.packages("e1071")
#install.packages("caret")
library("doParallel")
library("randomForest")
library("e1071")
library("caret")
suppressWarnings(suppressMessages(library(caret)))
suppressWarnings(suppressMessages(library(randomForest)))
suppressWarnings(suppressMessages(library(e1071)))

trainingFilename   <- 'pml-training.csv'
quizFilename       <- 'pml-testing.csv'
trainingUrl        <- 'https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv'
quizUrl            <- 'https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv'

download.file(trainingUrl, trainingFilename)
download.file(quizUrl,quizFilename)

```

###Data Cleansing

I inspected the data in excel and found a lot of NA, divide by zero, and blank values in the data. So I removed these values.
```{R Model 1, cache = TRUE}
training.df     <-read.csv(trainingFilename, #na.strings=c("NA","","#DIV/0!"))
training.df     <-training.df[,colSums(is.na(training.df)) == 0]
dim(training.df) #;head(training.df,3)
quiz.df         <-read.csv(quizFilename , na.strings=c("NA", "", #"#DIV/0!"))
quiz.df         <-quiz.df[,colSums(is.na(quiz.df)) == 0]
dim(quiz.df) #;head(quiz.df,3)
```

##Reduce The number of variables and look for near-zero values in training set

I then went abbout removing the non-predictors from the training set. These include the index, subject name, time, and window variables.

```{r Model 2, cache = TRUE}
Training.df   <-training.df[,-c(1:7)]
Quiz.df <-quiz.df[,-c(1:7)]
dim(Training.df)
Training.nzv<-nzv(Training.df[,-ncol(Training.df)],saveMetrics=TRUE)
rownames(Training.nzv)
dim(Training.nzv)[1]

```

# Develop an algorithm by partitioning the training data into a training and then a test/validation set.

```{R Algorithm, cache = TRUE}
inTrain     <- createDataPartition(Training.df$classe, p = 0.6, list = FALSE)
inTraining  <- Training.df[inTrain,]
inTest      <- Training.df[-inTrain,]
dim(inTraining);dim(inTest)

 myModelFilename <- "myModel.RData"
if (!file.exists(myModelFilename)) {

    # Parallel cores  
    #require(parallel)
    library(doParallel)
    ncores <- makeCluster(detectCores() - 1)
    registerDoParallel(cores=ncores)
    getDoParWorkers() # 3    
    
    # use Random Forest method with Cross Validation, 4 folds
    myModel <- train(classe ~ .
                , data = inTraining
                , method = "rf"
                , metric = "Accuracy"  # categorical outcome variable so choose accuracy
                , preProcess=c("center", "scale") # attempt to improve accuracy by normalising
                , trControl=trainControl(method = "cv"
                                        , number = 4 # folds of the training data
                                        , p= 0.60
                                        , allowParallel = TRUE 
                                       , seeds=NA # don't let workers set seed 
                                        )
                )

    save(myModel, file = "myModel.RData")

    stopCluster(ncores)
}# else {
    # Use cached model  
load(file = myModelFilename, verbose = TRUE)
}
```


## Results and Conclusion

We have constructed a model and have achieved cross validation by using the trainControl method set to 'cv'

```{R Validation}
print(myModel, digits=4)

```


## Predict, Evaluate, and Test with a confusion matrix.

```{r prediction, echo=FALSE}
predTest <- predict(myModel, newdata=inTest)

confusionMatrix(predTest, as.factor(inTest$classe))
```
The out of sample error is 0.14%. The accuracy is quite high at 99.46%. This is definitely within a 95% confidence interval.


## Create a final model for data and all important predictors.

```{r model final, echo=FALSE}
myModel$finalModel
print(predict(myModel, newdata=Quiz.df))
```


```{r model final var, echo=FALSE}
varImp(myModel)
print(predict(myModel, newdata=Quiz.df))
```

27 variables were tested at each split and the reported out of bounds estimated error is quite low at 0.83%. We have sufficient confidence in the prediction model for the 20 test cases.

---
---