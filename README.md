Reproducible-Research---Peer-Assessment-2-
==========================================
---
title: "How severe weather events are harmful to population health" 
author: "Leandro Guerra"
date: "Saturday, August 23, 2014"
---

##Synopsis 

The goal of this report is show the most harmful events that can be affect the united states, in order to take some decisions related to the future investments in planning and management of damages. The report identifies the most significant weather event types with the largest impact on population health (as measured by mean number of combined fatalities and injuries) and the largest economic consequences (as measured by the mean property damage and mean crop damage sustained during the event). This analysis finds that it is wind-related events such as tropical storms and tornadoes that have the greatest impact on population health and cause the most property damage. Excessive wetness and temperature extremes are the events that cause the most severe crop damage.

**About the Data**

The weather events are divided into 13 groups:

-Convection (e.g. tornado, lightning, thunderstorm, hail)
-Flood (e.g. flash flood, river flood)
-Extreme temperatures (e.g. extreme cold, extreme hot)
-Marine (e.g. tsunami, coastal storm, rip current, high waves, high seas)
-Winter (e.g. avalanche, snow, blizzard, icy roads, freeze)
-Tropical Cyclones (e.g. tropical storm, hurricane)
-High Wind (e.g. winds, microburst)
-Fire
-Rain
-Drought/Dust (e.g. drought, dust storm, dust)
-Landslide
-Fog
-Others

## Data Processing

```{r importdata, echo=TRUE}
#Setting WD
setwd("C:/Users/Leandro/Google Drive/Coursera/DATASCIENCE")

#Unzip and read .csv file into the variable data
unzip <- bzfile("repdata-data-StormData.csv.bz2", "r")
data <- read.csv(unzip, stringsAsFactors = FALSE)
close(unzip)
```

**Select useful data** 

Subsetting data into variables that are needed and adding a new variable.

```{r datasubset,echo=TRUE,cache=TRUE}
x <- which(colnames(data) %in% c("BGN_DATE", "PROPDMG", "CROPDMG", "EVTYPE", 
    "INJURIES", "FATALITIES"))
data <- data[, x]
head(data)

#Formatting date and time
data$YEAR <- as.integer(format(as.Date(data$BGN_DATE, "%m/%d/%Y 0:00:00"), "%Y"))
head(data)

#To uppercase
data$EVTYPE <- toupper(data$EVTYPE)
head(data)

# creates new variable
data$ECONOMICDMG <- data$PROPDMG + data$CROPDMG
head(data)

# Select only positive value data
data <- subset(data, data$FATALITIES > 0 | data$ECONOMICDMG > 0 | data$INJURIES > 
    0)
head(data)
```

Data aggregation

```{r aggregate,echo=TRUE,cache=TRUE}
library(plyr)

# data aggregated by YEAR & EVTYPE.
#ddply -> For each subset of a data frame, apply function then combine results into a data frame.

eventYear <- ddply(data[, -1], .(YEAR, EVTYPE),
                   .fun = function(x) {
                         return(
                           c(sum(x$FATALITIES), sum(x$ECONOMICDMG), sum(x$INJURIES))
                              )
                                      }
                   )
names(eventYear) <- c("YEAR", "EVTYPE", "FATALITIES", "ECONOMICDMG", "INJURIES")
head(eventYear)
```

**Grouping the events**
We grouped the events by its related categories

```{r group,echo=TRUE,cache=TRUE}
#Function that calculates the events by categories (13 categories described in the synopsis)

#grepl -> search for matches to argument pattern within each element of a character vector

eventCategory <- function(x) {
    ev <- x$EVTYPE[1]
    if (grepl("LIG(H|N)T(N|)ING|TORNADO|T(H|)U(N|)(DER|ER|DEER|DERE)(STORM|STROM|TORM)|TSTM|HAIL", 
        ev)) {
        category <- "Convection"
    } else if (grepl("WINT(ER|RY)|ICE|AVALANC(H|)E|SNOW|BLIZZARD|FREEZ|ICY|FROST", 
        ev)) {
        category <- "Winter"
    } else if (grepl("COLD|HEAT|HOT|TEMPERATURE|COOL|WARM", ev)) {
        category <- "Extreme Temp"
    } else if (grepl("FLOOD| FLD$", ev)) {
        category <- "Flood"
    } else if (grepl("COASTAL|TSUNAMI|RIP CURRENT|MARINE|WATERSPOUT|SURF|SLEET|SEAS|(HIGH|RISING|HEAVY) (WAVES|SWELLS|WATER)", 
        ev)) {
        category <- "Marine"
    } else if (grepl("TROPICAL|HURRICANE|STORM SURGE|TYPHOON", ev)) {
        category <- "Tropical Cyclones"
    } else if (grepl("WIND|MICROBURST", ev)) {
        category <- "High Wind"
    } else if (grepl("FIRE", ev)) {
        category <- "Fire"
    } else if (grepl("RAIN|PRECIP", ev)) {
        category <- "Rain"
    } else if (grepl("DROUGHT|DUST", ev)) {
        category <- "Drought/Dust"
    } else if (grepl("LANDSLIDE|MUD.*SLIDE", ev)) {
        category <- "Landslide"
    } else if (grepl("FOG|VOG", ev)) {
        category <- "Fog"
    } else {
        category <- "Others"
    }

    x$EVGROUP <- rep(category, dim(x)[1])
    return(x)
}
eventYear <- ddply(eventYear, .(EVTYPE), .fun = eventCategory)
head(eventYear)

#We organize the data to show FATALITIES, ECONOMICDMG and INJURIES
#by YEAR and EVGROUP

groupYear <- ddply(eventYear, .(YEAR, EVGROUP), .fun = function(x) {
    return(c(sum(x$FATALITIES), sum(x$ECONOMICDMG), sum(x$INJURIES)))
})

names(groupYear) <- c("YEAR", "EVGROUP", "FATALITIES", "ECONOMICDMG", "INJURIES")
head(groupYear)

# calculate average annual damage by group
eventFirstYear <- ddply(groupYear, .(EVGROUP), .fun = function(x) {
    return(c(min(x$YEAR)))
})
names(eventFirstYear) <- c("Weather.Event", "First.Year")
head(eventFirstYear)
```

As we can notice analysing the variable eventFirstYear, the weather event "Convection" has its occurency starting at the 50's but the others events starts at 1993. In this section we subset the groupYear to analysis all the events starting from 1993

```{r data1993,echo=TRUE,cache=TRUE}

## start data analysis at 1993
groupYear <- subset(groupYear, YEAR >= 1993)

# calculate average annual damage by group
byGroup <- ddply(groupYear, .(EVGROUP), .fun = function(x) {
    return(c(mean(x$FATALITIES), mean(x$ECONOMICDMG), mean(x$INJURIES)))
})
names(byGroup) <- c("EVGROUP", "AVG.FATALITIES", "AVG.ECONOMICDMG", "AVG.INJURIES")
head(byGroup)

```

## Results

**Results section 1 - Health Harmful Events**

This histograms Show fatalities and injuries for weather events.

```{r results1,echo=TRUE, cache=TRUE}
# Graph libraries
library(ggplot2)
library(scales)

# average annual populational damage by group of event
byGroup$EVGROUP <- with(byGroup, reorder(EVGROUP, -AVG.FATALITIES))
g <- ggplot(byGroup, aes(x = EVGROUP))
g + geom_histogram(aes(weight = AVG.FATALITIES, fill = AVG.FATALITIES), binwidth = 5, 
    color = "black") + ggtitle("Average Fatalities") + ylab("# Fatalities") + 
    xlab("Weather Event") + theme(axis.text.x = element_text(angle = 45, hjust = 1))

# average annual populational damage by group of event
byGroup$EVGROUP <- with(byGroup, reorder(EVGROUP, -AVG.INJURIES))
g <- ggplot(byGroup, aes(x = EVGROUP))
g + geom_histogram(aes(weight = AVG.INJURIES, fill = AVG.INJURIES), binwidth = 1, 
    color = "black") + ggtitle("Average Injuries") + ylab("# Injuries") + xlab("Weather Event") + 
    theme(axis.text.x = element_text(angle = 45, hjust = 1))
```


**Results section 2 - Economic Harm**
  
Histogram of weather event harm to the economy.

```{r results2,echo=TRUE, cache=TRUE}
# average annual economical damage by group of event
byGroup$EVGROUP <- with(byGroup, reorder(EVGROUP, -AVG.ECONOMICDMG))
g <- ggplot(byGroup, aes(x = EVGROUP))
g + geom_histogram(aes(weight = AVG.ECONOMICDMG, fill = AVG.ECONOMICDMG), binwidth = 1, 
    color = "black") + ggtitle("Average Economic Damage") + ylab("Economic damage") + 
    xlab("Weather Event") + theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

## Conclusion 

According to the analysis, we can notice in the results that the most harmful events for population are "Extreme temperatures" and "Convection" when we look at "Average Fatalities". When we talk about "Average Injuries", we have the same events, but in a different order - "Convection" and "Extreme Temperatures".

Now, when we look at economic damage,the extremely harmful events for economy are
"Convection" and "Flood". It is quite logical think about it, but here we can prove with data.

