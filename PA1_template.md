---
title: "Peer Assessment 1: Reproducible Research"
author: "irv"
date: "Sunday, September 20, 2015"
output: html_document
---
##Introduction##

The problem presented was to perform simple analysis on movement data from a simple health tracking device such as a FitBit or similar item. Data provided (in csv format) gives a count of steps walked in the previous 5 minutes over a period of two months.

This document describes questions asked about the activities shown by the data and the methods used to answer them.


##Assumptions##

1. The data file is in the same folder as this document
2. Required packages are available for loading. Those include:
  1. sqldf
  2. nnet
3. That's all. thanks for playing!

## Housekeeping ##

First, we're going to call the required packages and read the file.


```r
    # turn off annoying messages
    options(warn=-1)
    suppressMessages(library(sqldf))
    suppressMessages(library(nnet))
    library(sqldf)
    library(nnet)
    csv <- read.csv("activity.csv")
```
Next, we're going to pull row and column names that will be reused several times in this assignment.


```r
    dates <- unique(as.character(csv$date))
    intervals <- unique(csv$interval)
```

## Part 1 ##
### What is mean total number of steps taken per day? ###

For this portion of the process, the data will be rshaped into a form that seems to work a littl ebetter for the analysis (wide rather than long). Following tidy principles, there will be one observation per row.


```r
    csv_wide <- data.frame(matrix(ncol=length(dates), nrow=length(intervals)))
    rownames(csv_wide) <- intervals
    colnames(csv_wide) <- dates

    # This is a brute force reshape to make the data wide instead of long
    # I tried working with reshape, reshape2 and tidyr for a couple hours
    # and got nowhere, so I'm doing the simplistic method instead.
    # All my years as a programmer did not prepare me well for R!
    for (d in dates){
        # got a date. Sum every rsteps row in csv that matches
        term <- paste("SELECT steps from csv where date='", d, "'", sep="")
        dt <- sqldf(term)
        csv_wide[,d] <- dt
    }
```

```
## Loading required package: tcltk
```

This gets us to the solutions portion of our program!


```r
    date_sums <- colSums(csv_wide, na.rm = TRUE)
    # It look slike the histogram might be flipped!
    # steps taken on the Y axis
    hist(date_sums, main="Total Steps Per Day", xlab="Number of Steps", ylab="Days occurred", col="yellow")
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png) 

```r
    # Store the mean and median for later use
    ma <- mean(date_sums)
    md <- median(date_sums)
```

The above histogram agrees with the calculated results:

- The mean number of steps per day is **9354.2295082.**
- The median number of steps per day is **10395**.

## Part 2 ##
### What is the average daily activity pattern? ###

The answer to this question can be derived from the data used above.


```r
    interval_means <- rowMeans(csv_wide, na.rm = TRUE)
    plot.ts(interval_means, type="l", ylab="Mean Steps/Day", xlab="Interval", axes=FALSE)
    
    # The plotter counts intervals instead of using the names
    # plus there isn't enough room to show them all, so, we'll generate a subset
    par(cex=0.7)
    xn <- seq(0, 13) * 25
    yn <- c(0, 50, 100, 150, 200)
    axis(1, at=xn, labels= as.character(xn * 5))
    axis(2, at=yn, labels=as.character(yn))
```

![plot of chunk unnamed-chunk-5](figure/unnamed-chunk-5-1.png) 

```r
    # repair
    par(cex=1.0)
```

The overall pattern is for a spike in activity early in the day, followed by an uneven dropoff until (as discussed above) the interval at 1380 minutes or so, when data becomes too spotty to analyze.

**Question:** Which 5-minute interval, on average across all the days in the dataset, contains the maximum number of steps?

The answer to this question is found in the interval means calculated above.


```r
    location_of_max <- which.is.max(interval_means)
```
**Answer:** Interval **835** is the one with the most steps, on average, across all days, with a total of **206.1698113**.

## Part 3 ##
### Imputing missing values ###

**Question:** How many missing values in the dataset?


```r
    missings <- is.na(csv$steps)
```

**Answer:** There are 2304 values missing from the dataset.

**Questions:** Devise a strategy for dealing with missing values and create a new dataset to implement it.

**Answer:** We can insert the previously calculated interval means in place of NAs, as follows:


```r
    # This function will be called once per row on the original dataset
    imputer <- function(row, means) {
        if (is.na(row["steps"])) {
            row["steps"] <- means[toString(row["interval"])]
        }
        
        row
    }

    # AGAIN, I tried to find a more elegant R-ish way to do this
    # but couldn't quite get it to work, and there's a deadline
    # Sooooo, brute force wins again:
    csv_imputed <- csv
    for (i in 1:length(csv_imputed$steps)) {
        csv_imputed[i,] <- imputer(csv_imputed[i,], interval_means)
    }
    
    # NOW reshape the data the way it was done in part 1
    # and calculate and display the same things
    # we'll reuse the intervals and dates from before
    csvi_wide <- data.frame(matrix(ncol=length(dates), nrow=length(intervals)))
    colnames(csvi_wide) <- dates
    rownames(csvi_wide) <- intervals

    for (d in dates){
        # got a date. Sum every rsteps row in csv that matches
        term <- paste("SELECT steps from csv_imputed where date='", d, "'", sep="")
        dt <- sqldf(term)
        csvi_wide[,d] <- dt
    }
    
    # do the calculations needed to finish this section:
    date_sums_i <- colSums(csvi_wide)  # remove NAs no longer needed
    hist(date_sums_i, main="Steps per day", xlab="Steps", ylab="Days", 
                      cex=0.7, col="bisque", axes=FALSE)
    xax <- c(0, 5000, 10000, 15000, 20000, 25000)
    yax <- c(0, 5, 10, 15, 20, 25, 30, 35)
    par(cex=0.8)
    axis(1, at=xax, labels=as.character(xax))
    axis(2, at=yax, labels=as.character(yax))
```

![plot of chunk unnamed-chunk-8](figure/unnamed-chunk-8-1.png) 

```r
    ma_i <- mean(date_sums_i)
    md_i <- median(date_sums_i)
    # repair:
    par(cex=1.0)
```

**Question:** How do the mean and median from this calculation compare with the ones done previously (before missing values were replaced)?

- The mean number of steps per day from the imputed data is 10766.19 which is a little higher from the previous value of 9354.2295082.
- The median number of steps per day for the imputed data is 10766.19 which is higher than the previously calculated median of 10395 but identical to the mean, (which makes perfect sense since daily means were substituted for the missing data).

## Part 4 ##
### Are there differences in activity patterns between weekdays and weekends? ###

The following function will help create the needed factor variable that we'll use to separate weekends from weekdays.


```r
    weekend_or_not <- function(date_string){
        day <- weekdays(as.Date(date_string))
        which_day = "weekday"
        if (day == "Saturday" || day == "Sunday") {
            which_day = "weekend"
        }
        which_day
    }
```

Now to reshape the imputed dataset into a form that will make answering the question simple.


```r
    new_f <- data.frame(matrix(nrow=length(dates), ncol=length(intervals)))
    rownames(new_f) <- dates
    colnames(new_f) <- intervals
    factor_col <- c()
    for (d in dates) {
        term <- paste("SELECT steps from csv_imputed where date='", d, "'", sep="")
        new_col <- sqldf(term)
        
        # new_row is actually a data frame, so access it this way
        new_f[d,] <- new_col[,1]
        factor_col <- c(factor_col, weekend_or_not(d))
    }
    
    # have to add the factors all at once
    # or else R coerces all the values in the column to chraracter
    # This way keeps them as numbers
    new_f$day_type <- factor(factor_col, levels=c("weekend", "weekday"))
```
And display the plots.


```r
weekend_data <- subset(new_f, day_type == "weekend", select=seq(1:288))
weekday_data <- subset(new_f, day_type == "weekday", select=seq(1:288))

# and now the plot. Sticking with standard plot, even though it's not as pretty. 
# I get better results when I keep it simple.
# Though I'm giving up the title background color from the example.
par(mfrow=c(2,1), mar=c(1,4,1,1), cex=0.75)
label_sz = 1.2

yx1 <- c(0, 500, 1000, 1500, 2000, 2500)
plot.ts(colnames(weekend_data), colSums(weekend_data), type="l", axes=FALSE, xlab="", ylab="")
axis(2, at=yx1, labels=as.character(yx1))
title(main="weekend", ylab="Number of Steps", cex.lab=label_sz)

yx2 <- c(1000,2000, 3000, 4000, 5000, 6000, 7000, 8000, 9000, 10000)
xx2 <- c(0, 500, 1000, 1500, 2000, 2500)
par(mar=c(4,4,1,1))
plot.ts(colnames(weekday_data), colSums(weekday_data), type="l", axes=FALSE, xlab="", ylab="")
axis(2, at=yx2, labels=as.character(yx2))
axis(1, at=xx2, labels=as.character(xx2))
title(main="weekday")
title(xlab="Interval", cex.lab=label_sz, ylab="Number of Steps")
```

<img src="figure/unnamed-chunk-11-1.png" title="plot of chunk unnamed-chunk-11" alt="plot of chunk unnamed-chunk-11" style="display: block; margin: auto;" />

**Note** The differing scale for the weekday plot as well as the differing patterns. while weekends seem to show a more sustained pattern of activity over the course of a day, there are about 4 times as many steps taken on weekdays than on weekends, indicating that the overall acitivity level on weekdays is much higher!

### Thus endeth the assignment ###



