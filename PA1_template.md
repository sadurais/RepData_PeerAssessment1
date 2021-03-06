---
title: "Reproducible Research: Peer Assessment 1"
author: "Sathish Duraisamy"
date: "January 18, 2015"
output: 
  html_document:
    keep_md: true
---

## Introduction
We have a data file activity.zip that contains the data generated by
wearable device during human motion walking steps. The objective is to 
answer the following question from the data file that we have.


## Loading and preprocessing the data
The data is loaded from the 'activity.csv' file from current directory into 
a data frame. A quick 'summary' and 'str' functions on this data frame show 
that the 'interval' variable needs to be reformatted correctly. The following
function reads the data, adds a new variable 'int_mins' to capture the 
interval as numerical minutes.


```r
library(stringr)
library(ggplot2)
library(scales)
library(stringr)


# Reads the activity.csv file, cleans up data,
# and returns a tidy data frame with imputed values
# for NA values of 'steps'
readData <- function(sanitizeAndImpute=TRUE) {
    
    zipFile <- "activity.zip"
    csvFile <- "activity.csv"
    if (!file.exists(csvFile)) {
        if (file.exists(zipFile)) {
            unzip(zipFile, exdir = ".")
        } else {
            stop(paste("Required data file ", zipFile, " is missing", sep=""))
        }
    }
    
    if (!file.exists(csvFile)) {
        stop(paste("Required csv file ", csvFile, " is missing", sep=""))
    }
    
    df <- read.csv(csvFile, header=TRUE,
                         colClasses=c("numeric", "Date", "integer"))
    set.seed(121212)

    if (!sanitizeAndImpute) return(df)
    
    # Add a variable 'int_mins' to our data frame,
    # reformatting the 'interval' as numerical minutes *correctly*
    int_str <- sprintf("%04d", df$interval)
    hours <- as.integer(str_sub(int_str, 1, 2))
    mins <- as.integer(str_sub(int_str, 3, 4))
    df$int_mins <-  hours * 60 + mins

    # Add a factor variable 'day_type' to our data frame,
    # to track weekdays vs weekends
    df$day_type <- ifelse(weekdays(df$date) %in%
                            c("Sunday", "Saturday"), "weekend", "weekday")
    df$day_type <- as.factor(df$day_type)

    # Impute NA values in 'steps' with the mean-step-per-interval value
    # (calculated across all days of each unique interval). This is better
    # than using per-day mean because many samples are considered and the
    # bias this "overall" mean introduces must be minimal
    avg_steps_per_int <- tapply(df$steps, df$int_mins, mean, na.rm=T)
    df_int <- data.frame(avg_steps = avg_steps_per_int,
                         interval = as.numeric(names(avg_steps_per_int)))
    na_rows <- which(is.na(df$steps))
    for (row in na_rows) {
        intvl <- df$int_mins[row]
        df$steps[row] <- df_int$avg_steps[df_int$interval==intvl]
    }

    df
}
```


```r
# Handy function to convert a numerical minutes to HH:MM formated string
minutesToHHMM <- function(mins) {
    hour <- trunc(mins/60)
    min  <- mins %% 60
    return(sprintf("%02d:%02d", hour, min))
}
```


## Question 1: What is mean total number of steps taken per day?


```r
# Question 1:  What is mean total number of steps taken per day?
# For this part of the assignment, you can ignore the missing values in the dataset.
library(lattice)

df <- readData(sanitizeAndImpute=FALSE)
df <- df[complete.cases(df), ]  # Throw away NA values

tot_steps_per_day <- tapply(df$steps, df$date, sum)

#Calculate and report the mean and median total number of steps taken per day
summary(tot_steps_per_day)[3:4]  # Median 10760 and Mean 10770
```

```
## Median   Mean 
##  10760  10770
```

```r
# Make a histogram of the total number of steps taken each day.
histogram(tot_steps_per_day, xlab="Total Steps per day (NA values removed)",
          ylab="Frequency (ie. Percent of Total)")
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-1.png) 

```r
# Question 1 Part 2: Now lets impute the NA values and redo the hist again
df <- readData(sanitizeAndImpute=TRUE)
df <- df[complete.cases(df), ]  # Throw away NA values

tot_steps_per_day <- tapply(df$steps, df$date, sum)

#Calculate and report the mean and median total number of steps taken per day
summary(tot_steps_per_day)[3:4]  # Median and Mean
```

```
## Median   Mean 
##  10770  10770
```

```r
# Make a histogram of the total number of steps taken each day.
histogram(tot_steps_per_day, xlab="Total Steps per day (NA values imputed)",
          ylab="Frequency (ie. Percent of Total)")
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-2.png) 

Total steps taken per day (with NA values removed):
   Median = 10760 and Mean = 10770
Total steps taken per day (with NA values imputed):
   Median = 10770 and Mean = 10770
   
These can be confirmed from the two summary outputs listed above.



## Question 2: What is the average daily activity pattern?


```r
library(stringr)
library(ggplot2)
library(scales)
library(stringr)

act <- readData(sanitizeAndImpute=TRUE)

# Now that the NA values in 'steps' are imputed with mean values,
# lets recalculate the mean/intvl
avg_steps_per_int <- tapply(act$steps, act$int_mins, mean)
int_mins = as.numeric(names(avg_steps_per_int))
# We use a dummy-date 2000/01/01 00:00:01 as the 'start' of all our
# time-intervals. While plotting the x-asix, we'll hid this date part
# and show only the time-part. Since ISOdate() creates
dummy_date <- ISOdate(2000, 1, 1, 0, 0, 1, tz="")
df <- data.frame(steps_pi = avg_steps_per_int,
                 int_mins = int_mins,
                 int_tm   = dummy_date + (int_mins * 60),
                 int_str  = minutesToHHMM(int_mins))

ggplot(df, aes(int_tm, steps_pi)) +
    geom_line() +
    scale_x_datetime(labels = date_format(format="%H:%M"), breaks = "2 hours") +
    xlab("Time Interval of day") +
    ylab("Mean Steps per time interval, across all days")
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png) 

```r
#Which 5-minute interval, on average across all the days in the dataset,
#contains the maximum number of steps?
max_row <- df$steps_pi == max(df$steps_pi)
result <- df[max_row, c("steps_pi", "int_str")]
result
```

```
##     steps_pi int_str
## 515 206.1698   08:35
```

As can be seen from the plot and also from the result output above, the
maximum mean steps per interval (measured across all day) is 206 steps
and the time interval is 08:35 AM.
The plot also shows overall activity between 6:00 AM to 9:00 PM and
less activiy at the rest of the hours, understandably the nap time.




## Imputing missing values
For imputing the NA values of 'steps' variable in our data set, we will 
use the strategy of filling them with the mean number of steps (measured
per interval, across all days). This is done within the readData() function
listed above.

The specific code that does this imputation:

```r
    avg_steps_per_int <- tapply(df$steps, df$int_mins, mean, na.rm=T)
    df_int <- data.frame(avg_steps = avg_steps_per_int,
                         interval = as.numeric(names(avg_steps_per_int)))
    na_rows <- which(is.na(df$steps))
    for (row in na_rows) {
        intvl <- df$int_mins[row]
        df$steps[row] <- df_int$avg_steps[df_int$interval==intvl]
    }
```


## Question 3: Are there differences in activity patterns between weekdays and weekends?

```r
library(stringr)
library(ggplot2)
library(scales)
library(stringr)

group_by_interval <- function(act) {

    # Now that the NA values in 'steps' are imputed with mean values,
    # lets recalculate the mean/intvl
    avg_steps_per_int <- tapply(act$steps, act$int_mins, mean)
    int_mins = as.numeric(names(avg_steps_per_int))
    # We use a dummy-date 2000/01/01 00:00:01 as the 'start' of all our
    # time-intervals. While plotting the x-asix, we'll hid this date part
    # and show only the time-part. Since ISOdate() creates
    dummy_date <- ISOdate(2000, 1, 1, 0, 0, 1, tz="")
    df <- data.frame(steps_pi = avg_steps_per_int,
                 int_mins = int_mins,
                 int_tm   = dummy_date + (int_mins * 60),
                 int_str  = minutesToHHMM(int_mins))
    df
}


act <- readData(sanitizeAndImpute=TRUE)
weekday_rows <- act$day_type == "weekday"
act_wd <- act[weekday_rows, ]
act_we <- act[!weekday_rows, ]

df <- group_by_interval(act_wd); df$day_type <- "weekday"
result_weekday <- df[df$steps_pi == max(df$steps_pi), c("steps_pi", "int_str")]
df2 <- group_by_interval(act_we); df2$day_type <- "weekend"
result_weekend <- df2[df2$steps_pi == max(df2$steps_pi), c("steps_pi", "int_str")]

df <- rbind(df, df2)

ggplot(df, aes(int_tm, steps_pi)) +
    geom_line(color="blue") +
    facet_wrap(~day_type, ncol=1) +
    scale_x_datetime(labels = date_format(format="%H:%M"), breaks = "2 hours") +
    xlab("Time Interval of day") +
    ylab("Mean Steps per time interval, across all days")
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6-1.png) 

```r
#Which 5-minute interval, on average across all the days in the dataset,
#contains the maximum number of steps?
print("Max mean steps/interval and the interval for weekdays: ")
```

```
## [1] "Max mean steps/interval and the interval for weekdays: "
```

```r
result_weekday
```

```
##     steps_pi int_str
## 515 230.3782   08:35
```

```r
print("Max mean steps/interval and the interval for weekdends: ")
```

```
## [1] "Max mean steps/interval and the interval for weekdends: "
```

```r
result_weekend
```

```
##     steps_pi int_str
## 555 166.6392   09:15
```

It is evident from the data that the activity spike is low during weekends and also the beginning of the activity starts a little later in the morning compared to weekdays. 

Weekdays:  Max mean steps/interval = 230    and the interval = 8:35 AM
Weekends:  Max mean steps/interval = 167    and the interval = 9:15 AM

--end--
