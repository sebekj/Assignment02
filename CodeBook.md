---
title: "Code Book"
author: "Jiri J. Sebek"
date: "2025-03-12"
output:
  pdf_document:
    latex_engine: xelatex
header-includes:
  - \usepackage{fvextra} % Required for advanced verbatim environments
  - \usepackage{fancyvrb} % Enhances verbatim text (code chunks)
  - \usepackage{upquote} % Ensures correct quote formatting in code
  - \usepackage{xcolor} % Required for syntax highlighting
  - \DefineVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}
  - \RecustomVerbatimEnvironment{Highlighting}{Verbatim}{breaklines,commandchars=\\\{\}}
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

# Assignment

## Full Data Description

<http://archive.ics.uci.edu/ml/datasets/Human+Activity+Recognition+Using+Smartphones>

## Assignment Data

<https://d396qusza40orc.cloudfront.net/getdata%2Fprojectfiles%2FUCI%20HAR%20Dataset.zip>

## Assignment

1.  You should create one R script called run_analysis.R that does the following.
2.  Merges the training and the test sets to create one data set.
3.  Extracts only the measurements on the mean and standard deviation for each measurement.
4.  Uses descriptive activity names to name the activities in the data set.
5.  Appropriately labels the data set with descriptive variable names.
6.  From the data set in step 4, creates a second, independent tidy data set with the average of each variable for each activity and each subject.

## Download data set, unzip, root directory

```{r}
url <- "https://d396qusza40orc.cloudfront.net/getdata%2Fprojectfiles%2FUCI%20HAR%20Dataset.zip"
download.file(url, destfile = "dataset.zip")
time_stamp <- Sys.time()
unzip("dataset.zip")
formatted_time <- format(time_stamp, "%Y-%m-%d %H:%M:%S")
write(x = formatted_time, file = "The time stamp for dataset download.txt")
```

## Load Libraries

```{r}
# Initialize
# Function to check, install, and load a package
install_and_load <- function(package_name) {
  if (!requireNamespace(package_name, quietly = TRUE)) {
    install.packages(package_name, dependencies = TRUE)
    message(paste("Package", package_name, "installed"))
  }
  suppressMessages(library(package_name, character.only = TRUE))
  message(paste("Package", package_name, "loaded"))
}

# Execution
# install_and_load("readr")
install_and_load("dplyr")
```

## Group-load data to R

```{r}
(function(){
# List all .txt files in the test directory (full path provided)
txt_files <- list.files("UCI HAR Dataset/test", pattern = "\\.txt$", full.names = TRUE)

# Loop through each file
for (file in txt_files) {
  # Extract file name without the directory and extension
  file_name <- tools::file_path_sans_ext(basename(file))
  
  # Read the file; adjust header and other parameters according to your file format
  data <- read.csv(file, header = FALSE)
  
  # Assign the data to an object with the same name as the file (without .txt)
  assign(file_name, data, envir = .GlobalEnv)
}
})()

(function(){
# List all .txt files in the train directory (full path provided)
txt_files <- list.files("UCI HAR Dataset/train", pattern = "\\.txt$", full.names = TRUE)

# Loop through each file
for (file in txt_files) {
  # Extract file name without the directory and extension
  file_name <- tools::file_path_sans_ext(basename(file))
  
  # Read the file; adjust header and other parameters according to your file format
  data <- read.csv(file, header = FALSE)
  
  # Assign the data to an object with the same name as the file (without .txt)
  assign(file_name, data, envir = .GlobalEnv)
}
})()
```

## Load Features (Variable Names), Activity Labels

```{r}
features <- read.table(file = "UCI HAR Dataset/features.txt")
activity_labels <- read.table(file = "UCI HAR Dataset/activity_labels.txt")
```

## Parsing the X_test

```{r}
# Parsing each line at every single space
test_x_parsed <- lapply(X_test[1], function(x){
  strsplit(trimws(x), "\\s+")
})

# Bumping up objects by one level (name of column)
test_x_parsed <- test_x_parsed[[1]] 

# Converting to numeric
test_x_parsed <- lapply(test_x_parsed[1:length(test_x_parsed)], as.numeric)
```

## Parsing the X_train

```{r}
# Parsing each line at every single space
train_x_parsed <- lapply(X_train[1], function(x){
  strsplit(trimws(x), "\\s+")
})

# Bumping up objects by one level (name of column)
train_x_parsed <- train_x_parsed[[1]] 

# Converting to numeric
train_x_parsed <- lapply(train_x_parsed[1:length(train_x_parsed)], as.numeric)
```

## ASSIGNMENT 1: Merge Test and Train data to create one data set; TEST FIRST

```{r}
merged_list <- c(test_x_parsed, train_x_parsed)
```

## Convert to data.frame

```{r}
merged_df <- as.data.frame(do.call(rbind, merged_list))
# colnames(merged_df) <- c(...) # Add feature/variable names
```

## ASSIGNMENT 2: Extract summary statistics for each feature

```{r}
summary <- cbind(mean = apply(X = merged_df, MARGIN = 2, FUN = mean), sd = apply(X = merged_df, MARGIN = 2, FUN = sd))

summary <- data.frame(summary)

summary <- cbind(feature = features[,2], summary)
```

## ASSIGNMENT 3: Use descriptive activity names to name activities

### Tidy activity labels

```{r}
activity_labels[,2] <- as.factor(activity_labels[,2])
names(activity_labels) <- c("key", "activity")

# Merge test data with labels
names(y_test) <- "key"
merged_test_labels <- merge(y_test, activity_labels[, c("key", "activity")], by = "key", all.x = TRUE)

# Merge train data with labels
names(y_train) <- "key"
merged_train_labels <- merge(y_train, activity_labels[, c("key", "activity")], by = "key", all.x = TRUE)

# Combine test and train data labels, TEST DATA FIRST
merged_labels <- rbind(merged_test_labels, merged_train_labels)
```

### Merge keys for subjects, TEST SUBJECT FIRST

```{r}
merged_subject <- c(subject_test[,1], subject_train[,1])
```

### Merge all data with activity labels and subjects' keys

```{r}
merged_df_all <- cbind(merged_df, merged_labels, subject = merged_subject)
```

## ASSIGNMENT 4: Label data with descriptive variable names

### Tidy feature names

```{r}
# Default replacements, retain readability
variables <- make.names(names = features[,2], unique = TRUE)

length(unique(variables))

# Trimming "\.$" to improve readability
variables <- sub("\\.$", "", variables)

length(unique(variables)) # Check whether all names are unique

# Substituting multiple dots for one underscore to improve readability
variables <- gsub("\\.{2,}", ".", variables)
variables <- gsub("\\.", "_", variables)

length(unique(variables)) # Check whether all names are unique
```

### Add unique column names/variable names/features to data

```{r}
names(merged_df_all) <- c(variables, c("key", "activity", "subject"))
```

## ASSIGNMENT 5: From data in 4, create a second, independent tidy data set with the average of each variable for each activity and each subject

```{r}
# Using dplyr
tidy_data <- merged_df_all %>%
  group_by(subject, activity) %>%
  summarise(across(1:561, mean), .groups = "drop")

head(tidy_data)
```

## Save the tidy data set

```{r}
write.csv(x = tidy_data, file = "tidy_data.txt", row.names = FALSE)
```