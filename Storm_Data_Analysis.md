---
title: "An Analysis of the Damage Caused by Storms"
author: "Raymond Dineen"
date: "8/14/2020"
output: 
  html_document: 
    keep_md: yes
---

### Synopsis

### Data Processing

The data comes in .csv format that is compressed using the bzip2 algorithm. It can
be read into R by simply using read.csv(), as shown below.


```r
URL <- "https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"
download.file(URL, destfile = "./StormData.csv.bz2")
stormData <- read.csv("StormData.csv.bz2")
```

This data set contains data from 1950 to 2011. According to [NOAA](https://www.ncdc.noaa.gov/stormevents/details.jsp?type=eventtype),
the recording of all 48 different event types didn't start until January 1996. Because of this,
we will filter out everything before that point. This also will help account for
technological improvements to storm preparation and make the finding more relevant
to the present day.


```r
stormData$BGN_DATE <- as.Date(stormData$BGN_DATE, "%m/%d/%Y")
after1996 <- subset(stormData, BGN_DATE >= "1996-01-01")
```

It is important for us to clean up the columns describing the cost of property and crop damage.
The value is broken into a number and a multiplier such as "K", "M", "B", referring to
thousands, millions, or billions. We will create columns with a singular numeric values for this data.
There is also one entry with a "0" multiplier which we can ignore since the reported damage is also 0.


```r
#clean property damage
after1996$PROPDMG[after1996$PROPDMGEXP == "K"] <- after1996$PROPDMG[after1996$PROPDMGEXP == "K"]*10^3
after1996$PROPDMG[after1996$PROPDMGEXP == "M"] <- after1996$PROPDMG[after1996$PROPDMGEXP == "M"]*10^6
after1996$PROPDMG[after1996$PROPDMGEXP == "B"] <- after1996$PROPDMG[after1996$PROPDMGEXP == "B"]*10^9
#repeat for crop damage
after1996$CROPDMG[after1996$CROPDMGEXP == "K"] <- after1996$CROPDMG[after1996$CROPDMGEXP == "K"]*10^3
after1996$CROPDMG[after1996$CROPDMGEXP == "M"] <- after1996$CROPDMG[after1996$CROPDMGEXP == "M"]*10^6
after1996$CROPDMG[after1996$CROPDMGEXP == "B"] <- after1996$CROPDMG[after1996$CROPDMGEXP == "B"]*10^9
```

The other important column to clean up is the EVTYPE column. There official 48 event types are listed in section 2.1.1 of the
[Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf). However, there are currently far more than 48 event types in the data set as we can see from using unique() on the EVTYPE column.


```r
length(unique(after1996$EVTYPE))
```

```
## [1] 516
```

This is the result of typos and slightly different terms often being used than the official list of event types. We can start by standardizing the capitalization on the data and then looking at a table to list the events by frequency.


```r
#make EVTYPE all caps
after1996$EVTYPE <- toupper(after1996$EVTYPE)
#make a table
head(sort(table(after1996$EVTYPE), decreasing = TRUE), 18)
```

```
## 
##                     HAIL                TSTM WIND        THUNDERSTORM WIND 
##                   207715                   128664                    81403 
##              FLASH FLOOD                    FLOOD                  TORNADO 
##                    50999                    24248                    23154 
##                HIGH WIND               HEAVY SNOW                LIGHTNING 
##                    19909                    14000                    13203 
##               HEAVY RAIN             WINTER STORM           WINTER WEATHER 
##                    11528                    11317                     6987 
##         MARINE TSTM WIND             FUNNEL CLOUD MARINE THUNDERSTORM WIND 
##                     6175                     6063                     5812 
##              STRONG WIND     URBAN/SML STREAM FLD               WATERSPOUT 
##                     3564                     3392                     3390
```

We notice right away that two of the most frequent events, TSTM WIND and THUNDERSTORM WIND, are actually the same event listed two
ways. We also see URBAN/SML STREAM FLD listed separately from FLASH FLOOD. We can fix these as well as use a few more techniques to
group as many of the common typos into their proper groups as we can. The plan is to ignore most of the very infrequent event types that don't get picked up as they are relatively insignificant in the context of 650000+ events.


```r
library(stringr)
#fix TSTM WIND and MARINE THUNDERSTORM WIND
after1996$EVTYPE <- str_replace_all(after1996$EVTYPE, "^TSTM WIND$", "THUNDERSTORM WIND")
after1996$EVTYPE <- str_replace_all(after1996$EVTYPE, "^MARINE THUNDERSTORM WIND$", "MARINE TSTM WIND")
#fix URBAN/SML STREAM FLD
after1996$EVTYPE <- str_replace_all(after1996$EVTYPE, "^URBAN/SML STREAM FLD$", "FLASH FLOOD")
#fix WILD/FOREST FIRE
after1996$EVTYPE <- str_replace_all(after1996$EVTYPE, "^WILD/FOREST FIRE$", "WILDFIRE")
#group all WINTER WEATHER+ into WINTER WEATHER
after1996$EVTYPE[str_detect(after1996$EVTYPE, "WINTER WEATHER|WINTRY")] <- "WINTER WEATHER"
#group all HOT+ and WARM+ into HEAT
after1996$EVTYPE[str_detect(after1996$EVTYPE, "HOT|WARM")] <- "HEAT"
#TSTM WIND/HAIL to HAIL
after1996$EVTYPE[str_detect(after1996$EVTYPE, "TSTM WIND/HAIL")] <- "HAIL"
#all instances of SURF to HIGH SURF
after1996$EVTYPE[str_detect(after1996$EVTYPE, "SURF")] <- "HIGH SURF"
#all instances of EXTREME COLD to EXTREME COLD/WIND CHILL
after1996$EVTYPE[str_detect(after1996$EVTYPE, "EXTREME COLD")] <- "EXTREME COLD/WIND CHILL"
#SNOW to HEAVY SNOW, WIND to HIGH WIND
after1996$EVTYPE <- str_replace_all(after1996$EVTYPE, "^SNOW$", "HEAVY SNOW")
after1996$EVTYPE <- str_replace_all(after1996$EVTYPE, "^WIND$", "HIGH WIND")
#all instances of HURRICANE or TYPHOON to HURRICANE/TYPHOON
after1996$EVTYPE[str_detect(after1996$EVTYPE, "HURRICANE|TYPHOON")] <- "HURRICANE/TYPHOON"
#all instances of SLEET or FREEZING RAIN to SLEET
after1996$EVTYPE[str_detect(after1996$EVTYPE, "SLEET|FREEZING RAIN")] <- "SLEET"
```

After this cleaning, 99.5% of the recorded events now fall within the top 40 most frequent EVTYPES with at least 200 occurrences. We will pay attention to mostly just those moving forward into the analysis.


```r
library(dplyr)
top40names <- names(sort(table(after1996$EVTYPE), decreasing = TRUE))[1:40]
top40 <- after1996 %>% filter(EVTYPE %in% top40names)
```

### Results

##### Which events were most harmful with respect to population health?

We can answer this question by considering which events caused the most deaths and injuries on average when they occurred as well as total deaths and injuries across the entire time span of the data, 1996-2011. 


```r
library(tidyr)
library(ggpubr)
#Most Dangerous to health per event
mean_per_event <- top40 %>% mutate(EVTYPE = as.factor(EVTYPE)) %>%
    group_by(EVTYPE) %>%
    summarize(meanfatal = mean(FATALITIES), meaninjur = mean(INJURIES)) %>%
    filter(meanfatal > 0.2 | meaninjur > 0.2) %>%
    gather(key = "stat", value = "value", -EVTYPE) %>%
    ggplot(aes(EVTYPE, value, fill = stat)) +
    geom_col(position = "dodge") +
    theme(axis.text.x = element_text(angle = 90, hjust = 0.95, size = 8),
          legend.text = element_text(size = 8), legend.title = element_text(size = 8), title = element_text(size = 10)) +
    scale_fill_discrete(labels = c("Average Deaths", "Average Injuries")) +
    labs(x = "Event Type", y = "", title = "Deaths and Injuries Per Event 1996-2011", fill = "Legend")

#Most dangerous to health sums over the whole data set
sums <- top40 %>% mutate(EVTYPE = as.factor(EVTYPE)) %>%
    group_by(EVTYPE) %>%
    summarize(sumfatal = sum(FATALITIES), suminjur = sum(INJURIES)) %>%
    filter(sumfatal > 1000 | suminjur > 1000) %>%
    gather(key = "stat", value = "value", -EVTYPE) %>%
    ggplot(aes(EVTYPE, value, fill = stat)) +
    geom_col(position = "dodge") +
    theme(axis.text.x = element_text(angle = 90, hjust = 0.95, size = 8),
          legend.text = element_text(size = 8), legend.title = element_text(size = 8), title = element_text(size = 10)) +
    scale_fill_discrete(labels = c("Total Deaths", "Total Injuries")) +
    labs(x = "Event Type", y = "", title = "Total Deaths and Injuries 1996-2011", fill = "Legend")

ggarrange(mean_per_event, sums, ncol = 2, nrow = 1)
```

![](Storm_Data_Analysis_files/figure-html/healthplot-1.png)<!-- -->

We can see from these graphs that hurricanes and excessive heat cause, by far, the most injuries per event with excessive heat causing the most deaths per event. However, we can also see that tornadoes have caused far more total injuries than any other event with excessive heat, floods, lightning, and thunderstorm wind also posting relatively high total injuries. This is because the frequency of these events occurring is so high that even though on average they may not be as devastating as a hurricane, the damage still adds up to be much higher overall.


