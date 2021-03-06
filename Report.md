# Analysis on the impact of weather events in the USA on population and economy based on the NOAA Storm Database

By An Vo
Date: 4nd Mars 2017
Source code can be found at https://github.com/anvd/ReproducibleResearch_CourseProject2

## Reproducible Research: Course Project 2

### 1. Synopsis

The National Oceanic and Atmospheric Administration (NOAA) maintains a public database for storm event. The data contains the type of storm event, details like location, date, estimates for damage to property as well as the number of human victims of the storm. This report analyses the impact of weather events on the USA's population and economy.

According to the below analysis, the answer of two requested question is as follows:

Question 1: Across the United States, which types of events (as indicated in the EVTYPE variable) are most harmful with respect to population health?
Answer: Across the United States, tornado is the number one cause of injuries, followed by Thunderstorm & Lightning, Flooding & High Surf, Heat & Drought. While, in terms of fatalities, Heat & Drought is the number one cause followed by Tornado, Flooding & Surf.
are the most harmful for population health.

Question 2: Across the United States, which types of events have the greatest economic consequences?
Flooding & High Surf, Wind & Storm are the two major weather event types that cause the economic loss.



### 2. Data Processing
##### 2.1. Load libraries
Loading necessary libraries

```r
library(RCurl)
library(R.utils)
library(plyr)
library(reshape2)
library(ggplot2)
library(grid)
library(gridExtra)
library(scales)
```


##### 2.2. Load data, process and cache data

```r
existProcessedData <- TRUE
# check if ProcessedStormData.RData variable already exists
if(file.exists("./data/ProcessedStormData.RData")){
  load("./data/ProcessedStormData.RData")
  existProcessedData <- FALSE
}

if(existProcessedData){
  # create a data folder if necessary
  if(!file.exists("./data")){
    dir.create("./data")
  }
    
  # download data from the internet
  if(!file.exists("./data/StormData.csv.bz2")){
    fileUrl <- "https://d396qusza40orc.cloudfront.net/repdata/data/StormData.csv.bz2"
    destPath <- "./data/StormData.csv.bz2"
    binData <- getBinaryURL(fileUrl, ssl.verifypeer=0L, followlocation=1L)
    destFileHandle <- file(destPath, open="wb")
    writeBin(binData,destFileHandle)
    close(destFileHandle)
  }
    
  # unzip bz2 file into csv file
  if(!file.exists("./data/StormData.csv")){
    filePath <- "./data/StormData.csv.bz2"
    destPath <- "./data/StormData.csv"
    bunzip2(filePath, destPath, overwrite=TRUE, remove=FALSE)
  }
}
```


##### 2.3. Load the data
Read the .csv file

```r
if(existProcessedData){
  csvStormData <- read.csv("./data/StormData.csv")
}
```

##### 2.4. Remove unused colums
Only keep the necessary columns for the analysis: BGN_DATE, EVTYPE, FATALITIES, INJURIES, PROPDMG, PROPDMGEXP, CROPDMG, CROPDMGEXP

```r
if(existProcessedData) {
  neededColumns <- c("BGN_DATE", "EVTYPE", "FATALITIES", "INJURIES", "PROPDMG", "PROPDMGEXP", "CROPDMG", "CROPDMGEXP")
  reducedStormData <- csvStormData[, neededColumns]
  reducedStormData <- rename(reducedStormData, c("FATALITIES"="fatalities", "INJURIES"="injuries"))
}
```

##### 2.5. Refactor BGN_DATE and remove some earlier years since they have only a small amount of observations.

```r
if(existProcessedData) {
  # the number of observation to be removed
  totalNumberOfObservations <- nrow(reducedStormData)
  cutOffPercentage = 0.7
  cutOffObservations = round(totalNumberOfObservations * cutOffPercentage)
  
  # introduce year column
  reducedStormData$year = as.numeric(format(as.Date(reducedStormData$BGN_DATE, format = "%m/%d/%Y"), "%Y"))
  
  # create dataset with count per year, reverse the recordset, create running total 
  yearRecords <- count(reducedStormData, "year")
  yearRecords <- yearRecords[order(yearRecords$year, decreasing=TRUE), ]
  yearRecords$runningTotal = cumsum(yearRecords$freq)
  beginningYear <- min(yearRecords[yearRecords$runningTotal < cutOffObservations, 1])
  
  # reduce the dataset
  reducedStormData <- reducedStormData[reducedStormData$year >= beginningYear, ]
  endYear <- max(reducedStormData$year)
  
  # clean reducedStormData
  reducedStormData$BGN_DATE <- NULL
  rownames(reducedStormData) <- NULL
}
```

##### 2.6. Refactor EVTYPE into 11 levels
Group similar weather events into to 11 types.

```r
if(existProcessedData) {
  reducedStormData$damageSource <- NA
  
  reducedStormData[grepl("precipitation|rain|hail|drizzle|wet|percip|burst|depression|fog|wall cloud", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Precipitation & Fog"
  reducedStormData[grepl("wind|storm|wnd|hurricane|typhoon", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Wind & Storm"
  reducedStormData[grepl("slide|erosion|slump", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Landslide & Erosion"
  reducedStormData[grepl("warmth|warm|heat|dry|hot|drought|thermia|temperature record|record temperature|record high", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Heat & Drought"
  reducedStormData[grepl("cold|cool|ice|icy|frost|freeze|snow|winter|wintry|wintery|blizzard|chill|freezing|avalanche|glaze|sleet", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Snow & Ice"
  reducedStormData[grepl("flood|surf|blow-out|swells|fld|dam break", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Flooding & High Surf"
  reducedStormData[grepl("seas|high water|tide|tsunami|wave|current|marine|drowning", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "High seas"
  reducedStormData[grepl("dust|saharan", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Dust & Saharan winds"  
  reducedStormData[grepl("tstm|thunderstorm|lightning", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Thunderstorm & Lightning"
  reducedStormData[grepl("tornado|spout|funnel|whirlwind", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Tornado"
  reducedStormData[grepl("fire|smoke|volcanic", 
                         reducedStormData$EVTYPE, ignore.case = TRUE), "damageSource"] <- "Fire & Volcanic activity"
  
  # remove uncategorized records (damageSource == NA) & cast as factor
  reducedStormData <- reducedStormData[complete.cases(reducedStormData[, "damageSource"]), ]
  reducedStormData$damageSource <- as.factor(reducedStormData$damageSource)
  
  # clean reducedStormData
  reducedStormData$EVTYPE <- NULL
}
```

##### 2.7. Calculate the real damage values
Use PROPDMG, CROPDMG, PROPDMGEXP and CROPDMGEXP to calculate the damage values

```r
if(existProcessedData){
  # convert the PROPDMGEXP & CROPDMGEXP values to power of 10
  toTenPower <- function(x){
    if(is.numeric(x)) {
      x <- x
    }
    else if(grepl("h", x, ignore.case=TRUE)) {
      x <- 2
    }
    else if(grepl("k", x, ignore.case=TRUE)) {
      x <- 3
    }
    else if(grepl("m", x, ignore.case=TRUE)) {
      x <- 6
    }
    else if(grepl("b", x, ignore.case=TRUE)) {
      x <- 9
    }
    else if(x == "" || x == " "){
      x <- 0
    }
    else{
      x <- NA
    }
    x
  }
   
  # function to calculate the damage value
  calculateAmount <- function(num, exp){
    pow <- toTenPower(exp)
    if(is.numeric(num)){
      num <- num * (10 ^ pow)
    }
    
    if(!is.numeric(num)){
      num <- 0
    }
    
    num
  }
  
  # create new fields to store property, crop and total damage values
  reducedStormData$propDamage <- mapply(calculateAmount, reducedStormData$PROPDMG, reducedStormData$PROPDMGEXP)
  reducedStormData$cropDamage <- mapply(calculateAmount, reducedStormData$CROPDMG, reducedStormData$CROPDMGEXP)
  reducedStormData$damageTotal = reducedStormData$propDamage + reducedStormData$cropDamage
  
  # clean reducedStormData
  reducedStormData$PROPDMG <- NULL
  reducedStormData$PROPDMGEXP <- NULL
  reducedStormData$CROPDMG <- NULL
  reducedStormData$CROPDMGEXP <- NULL
}
```

##### 2.8. Prepare data for plotting

```r
if(existProcessedData){
  # aggregate economic damage by damageSource
  sumEconomicDamage <- aggregate(formula=cbind(propDamage, cropDamage, damageTotal) ~ damageSource, data=reducedStormData, FUN=sum, na.rm=TRUE)
  sumEconomicDamage <- sumEconomicDamage[order(sumEconomicDamage$damageTotal, decreasing=TRUE),]
  rownames(sumEconomicDamage) <- NULL
  sumEconomicDamage$damageSource <- factor(sumEconomicDamage$damageSource, levels=rev(sumEconomicDamage$damageSource))
  
  # melt the sumEconomicDamage into data frame to be used as bar chart
  meltSumEconomicDamage <- melt(sumEconomicDamage, id.vars=c("damageSource"), measure.vars=c("propDamage","cropDamage"), variable.name="damageType", value.name="damage")
  levels(meltSumEconomicDamage$damageType)[levels(meltSumEconomicDamage$damageType)=="propDamage"] <- "property"
   levels(meltSumEconomicDamage$damageType)[levels(meltSumEconomicDamage$damageType)=="cropDamage"] <- "crops"
  
  # aggregate human damage by damageSource
  sumHumanDamage <-aggregate(formula=cbind(injuries, fatalities) ~ damageSource, data=reducedStormData, FUN=sum, na.rm=TRUE) 
  sumHumanDamage <- sumHumanDamage[order(sumHumanDamage$injuries, decreasing=TRUE),]
  rownames(sumHumanDamage) <- NULL
  sumHumanDamage$damageSource <- factor(sumHumanDamage$damageSource, levels=rev(sumHumanDamage$damageSource))
  
  # define max values for bar chart scale
  maxInjuries <- max(sumHumanDamage$injuries)
  maxInjuries <- maxInjuries + round(maxInjuries * 0.25)
 
  maxFatalities <- max(sumHumanDamage$fatalities)
  maxFatalities <- maxFatalities + round(maxFatalities * 0.25)  
}
```

##### 2.9. Save data to RData file 
Save the processed data to an RData file

```r
if(existProcessedData){
  save(reducedStormData, 
       sumHumanDamage, 
       meltSumEconomicDamage,
       sumEconomicDamage, 
       maxInjuries, 
       maxFatalities,
       beginningYear,
       endYear,
       file="./data/ProcessedStormData.RData")
}
```


### 3. Results
##### 3.1. Injuries vs. Fatalities
Question 1: Across the United States, which types of events (as indicated in the EVTYPE variable) are most harmful with respect to population health?
Answer: Across the United States, tornado is the number one cause of injuries, followed by Thunderstorm & Lightning, Flooding & High Surf, Heat & Drought. While, in terms of fatalities, Heat & Drought is the number one cause followed by Tornado, Flooding & Surf.
are the most harmful for population health.

```r
# add middle column with just damageSource labels
g.mid <- ggplot(data=sumHumanDamage, aes(x=1, y=damageSource)) +
            geom_text(aes(label=damageSource), size=4) +
            ggtitle("") +
            ylab(NULL) +
            scale_x_continuous(expand=c(0,0), limits=c(0.94, 1.065)) +
            theme(axis.title=element_blank(),
                  panel.grid=element_blank(),
                  axis.text.y=element_blank(),
                  axis.ticks.y=element_blank(),
                  panel.background=element_blank(),
                  axis.text.x=element_text(color=NA),
                  axis.ticks.x=element_line(color=NA),
                  plot.margin = unit(c(1,-1,1,-1), "mm"))

# add left chart with injuries
g.injuries <- ggplot(data=sumHumanDamage, aes(x=damageSource, y=injuries)) +
            geom_bar(stat = "identity", fill= "yellow", colour= "black") + 
            geom_text(aes(label=injuries), size=3, vjust=0.5, hjust=2.0) +
            ggtitle("Injuries") +
            scale_y_reverse(expand=c(0, 0), limits=c(maxInjuries,0)) + 
            coord_flip() +
            theme(axis.title.x = element_blank(), 
                  axis.title.y = element_blank(), 
                  axis.text.y = element_blank(), 
                  axis.ticks.y = element_blank(), 
                  plot.margin = unit(c(1,-1,1,0), "mm")) 

# add right chart with fatalities
g.fatalities <- ggplot(data=sumHumanDamage, aes(x=damageSource, y=fatalities)) +
            geom_bar(stat = "identity", fill="red", colour="black") + 
            geom_text(aes(label=fatalities), size=3, vjust=0.5, hjust=-1.0) +
            ggtitle("Fatalities") +
            scale_y_continuous(expand=c(0, 0), limits=c(0, maxFatalities)) + 
            coord_flip() +
            theme(axis.title.x = element_blank(), 
                  axis.title.y = element_blank(), 
                  axis.text.y = element_blank(), 
                  axis.ticks.y = element_blank(), 
                  plot.margin = unit(c(1,0,1,-1), "mm")) 

# combine charts in one plot
gg.injuries <- ggplot_gtable(ggplot_build(g.injuries))
gg.fatalities <- ggplot_gtable(ggplot_build(g.fatalities))
gg.mid <- ggplot_gtable(ggplot_build(g.mid))

grid.arrange(gg.injuries,gg.mid,gg.fatalities,
             ncol=3, widths=c(4/10,2/10,4/10),
             top=paste("Aggregated human injuries & fatalities for weather events from ", beginningYear," to ", endYear, sep=""))
```

![](Report_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

##### 3.2. Economic Loss
Question 2: Across the United States, which types of events have the greatest economic consequences?
Flooding & High Surf, Wind & Storm are the two major weather event types that cause the economic loss.

```r
ggplot(meltSumEconomicDamage, aes(x=damageSource, y=damage/1000000)) + 
  geom_bar(stat = "identity", aes(fill=damageType)) +
  xlab("Event Type") +
  theme(axis.text.x = element_text(angle = 45, size=8, hjust = 1, vjust = 1)) +
  ylab("Total Loss (millions of USD)") +
  ggtitle(paste("Aggregated property and crop damage for weather events from ", beginningYear, " to ", endYear, sep=""))
```

![](Report_files/figure-html/unnamed-chunk-11-1.png)<!-- -->
