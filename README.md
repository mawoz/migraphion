# making relationship graphs
David Hood  
11 August 2015  

I am putting together a relationship graph from multiple sources. The sources are

__United Nations Statistics Division. Countries or areas, codes and abbreviations (last updated 6 Nov. 2013):__

http://unstats.un.org/unsd/methods/m49/m49alpha.htm

saved as a csv (UTF-8, unix(lf) line endings) called tla.csv in the data folder

__United Nations, Department of Economic and Social Affairs (2013). Trends in International Migrant Stock: Migrants by Destination and Origin (United Nations database, POP/DB/MIG/Stock/Rev.2013), table 10:__		

http://esa.un.org/unmigration/TIMSO2013/data/subsheets/UN_MigrantStockByOriginAndDestination_2013T10.xls

saved as a csv (UTF-88, unix(lf) line ending) called tab10.csv in the data folder

__United Nations, Department of Economic and Social Affairs, Population Division (2015). World Population Prospects: The 2015 Revision, DVD Edition. Total population- both sexes.__					

http://esa.un.org/unpd/wpp/DVD/Files/1_Excel%20(Standard)/EXCEL_FILES/1_Population/WPP2015_POP_F01_1_TOTAL_POPULATION_BOTH_SEXES.XLS

saved as a csv (UTF-88, unix(lf) line ending) called poptot.csv in the data folder

csv copies have been made as reading in data from xls files can be difficult to be consistent across platforms, particularly when dealing with unicode characters.

We going to need a bunch of packages installed in R, if you don't alread have them, this run through uses:


```r
#install.packages("circlize")
#install.packages("tidyr")
#install.packages("igraph")
library(tidyr)
library(igraph)
```

```
## 
## Attaching package: 'igraph'
## 
## The following objects are masked from 'package:stats':
## 
##     decompose, spectrum
## 
## The following object is masked from 'package:base':
## 
##     union
```

```r
library(circlize)
```

Read in data:

```r
mig <- read.csv("data/tab10.csv", stringsAsFactors=FALSE)
pop <-  read.csv("data/poptot.csv", stringsAsFactors=FALSE)
tla <-  read.csv("data/tla.csv", stringsAsFactors=FALSE)
```

Fix up the migration data into a form I find more usable, checking it on the way


```r
#make the names match the headings in mig (because the headings were made syntacticlly legal on reading in)
mig[,2] <- make.names(mig[,2]) #using 2 because name is looong at present
#confirm matches
unique(mig[,2])[!(unique(mig[,2]) %in% names(mig))]
```

```
##  [1] "WORLD"                                                     
##  [2] "More.developed.regions"                                    
##  [3] "Less.developed.regions"                                    
##  [4] "Least.developed.countries"                                 
##  [5] "Less.developed.regions.excluding.least.developed.countries"
##  [6] "Sub.Saharan.Africa"                                        
##  [7] "AFRICA"                                                    
##  [8] "Eastern.Africa"                                            
##  [9] "Middle.Africa"                                             
## [10] "Northern.Africa"                                           
## [11] "Southern.Africa"                                           
## [12] "Western.Africa"                                            
## [13] "ASIA"                                                      
## [14] "Central.Asia"                                              
## [15] "Eastern.Asia"                                              
## [16] "South.Eastern.Asia"                                        
## [17] "Southern.Asia"                                             
## [18] "Western.Asia"                                              
## [19] "EUROPE"                                                    
## [20] "Eastern.Europe"                                            
## [21] "Northern.Europe"                                           
## [22] "Southern.Europe"                                           
## [23] "Western.Europe"                                            
## [24] "LATIN.AMERICA.AND.THE.CARIBBEAN"                           
## [25] "Caribbean"                                                 
## [26] "Central.America"                                           
## [27] "South.America"                                             
## [28] "NORTHERN.AMERICA"                                          
## [29] "OCEANIA"                                                   
## [30] "Australia.and.New.Zealand"                                 
## [31] "Melanesia"                                                 
## [32] "Micronesia"                                                
## [33] "Polynesia"
```

```r
#only regions, and I will be working with countries and territories, so not a problem.
names(mig)[!(names(mig) %in% unique(mig[,2]))]
```

```
##  [1] "Sort..order"                                       
##  [2] "Major.area..region..country.or.area.of.destination"
##  [3] "Notes"                                             
##  [4] "Country.code"                                      
##  [5] "Type.of.data..a."                                  
##  [6] "X"                                                 
##  [7] "World"                                             
##  [8] "Other.North"                                       
##  [9] "Other.South"                                       
## [10] "X.1"                                               
## [11] "X.2"                                               
## [12] "X.3"                                               
## [13] "X.4"                                               
## [14] "X.5"                                               
## [15] "X.6"                                               
## [16] "X.7"                                               
## [17] "X.8"                                               
## [18] "X.9"                                               
## [19] "X.10"                                              
## [20] "X.11"                                              
## [21] "X.12"                                              
## [22] "X.13"                                              
## [23] "X.14"                                              
## [24] "X.15"
```

```r
#only regions and extra columns so not a worry
#save off the numers and names
names(mig)[2] <- "destinationto"
idnums <- mig[,c("destinationto","Country.code")]
names(idnums) <-c("regname","regnumber")
#pivot to the long form and tidy up the data
miglong <- gather(mig[,c(2,4,7:241)], sourcefrom, flow, -(destinationto:Country.code))
miglong$flow <- as.integer(gsub("[^1234567890]","",miglong$flow))
miglong$flow[is.na(miglong$flow)] <- 0
miglong$sourcefrom <- as.character(miglong$sourcefrom)
#apply country code numbers to sourcefrom
miglong <- merge(miglong, idnums, by.x = "sourcefrom", by.y = "regname")
#ones which did not have a match (so have no id number) are regions that are not of interest
miglong <- miglong[complete.cases(miglong),]
#only keep entries with ids below 900 (excludes regions) for the data and drop the names now we have ids
miglong <- miglong[miglong$Country.code < 900 & miglong$regnumber < 900 , c("Country.code","flow","regnumber")]
### we also don't need the from the same as the to, or the entries where the flow was zero
miglong <- miglong[miglong$Country.code != miglong$regnumber & miglong$flow > 0 ,]
names(miglong) <- c("destinationID", "flow","sourceID")
### add Three Letter Codes
miglong <- merge(miglong, tla[,c(1,3)], by.x="destinationID", by.y="numericCode")
names(miglong)[4] <- "distinationTLA"
miglong <- merge(miglong, tla[,c(1,3)], by.x="sourceID", by.y="numericCode")
names(miglong)[5] <- "sourceTLA"
```

Now we'll merge in the population information


```r
miglong <- merge(miglong, pop[,c("Country.code","X2013")], by.x="destinationID", by.y="Country.code")
names(miglong)[6] <- "destinationPop"
miglong <- merge(miglong, pop[,c("Country.code","X2013")], by.x="sourceID", by.y="Country.code")
names(miglong)[7] <- "sourcePop"
miglong$destinationPop <- as.integer(gsub("[^1234567890]","",miglong$destinationPop))
miglong$sourcePop <- as.integer(gsub("[^1234567890]","",miglong$sourcePop))
```

That should be all the data that is needed brought together in one place, but it would be a mistake to graph them all at once.

```r
library(igraph)
g <- graph.data.frame(miglong[,c(5,4)], directed=TRUE)
plot(g, vertex.size = 1, vertex.label.degree=0, vertex.color="#BBBBBB")
```

![](README_files/figure-html/unnamed-chunk-2-1.png) 

Whatever we did with it, with a static graph there are just too many edges (lines), as a few people from most countries come to each country which makes a few hundred lines for each country. We need to restrict the data to make it clearer, but we will get very different kinds of graph depending on the the way we restrict the data. We could restrict it to the strongest source country for each destination country.


```r
topflow <- aggregate(flow ~ destinationID, miglong, max)
names(topflow)[2] <- "maxflow"
maxmig <- merge(miglong,topflow)
maxmig <- maxmig[maxmig$flow == maxmig$maxflow,]
```

Now we take the first steps on a circular graph

```r
wat <- function(x){
  y <- tla$ISOalpha3Code[tla$numericCode == as.numeric(x)]
  return(y)
}
chordDiagram(maxmig[,c("sourceID","destinationID")], annotationTrack="grid", preAllocateTracks=list(track.height = 0.3))
circos.trackPlotRegion(track.index = 1, panel.fun=function(x,y){
  xlim=get.cell.meta.data("xlim")
  ylim=get.cell.meta.data("ylim")
  sector.name = wat(get.cell.meta.data("sector.index"))
  circos.text(mean(xlim),ylim[1], sector.name, facing="clockwise", niceFacing=TRUE, adj=c(0,0.5), cex=0.4)}, bg.border=NA)
```

![](README_files/figure-html/chordchart-1.png) 

Well, thats gorgeous, but I don't think it is the right kind of graph in this case, we really need to move around the countries to minimise the crossing lines.


```r
set.seed(23913)
g <- graph.data.frame(maxmig[,c("sourceTLA","distinationTLA")], directed=TRUE)
plot(g, vertex.size = 1, vertex.label.cex=0.5, vertex.color="#BBBBBB", edge.arrow.size=0.3, edge.color="#BBBBBB", vertex.frame.color="#BBBBBB", edge.width=1, vertex.label.color="black", vertex.label.dist=0.2, main="Major immigration source country for every country in the world", sub="Source: UN Migrant Stock By Origin And Destination 2013 table 10\n Country Codes: http://unstats.un.org/unsd/methods/m49/m49alpha.htm")
```

![](README_files/figure-html/igraphchart-1.png) 

The set.seed controls the random elements in the graph generation, so it can be experimented with to get a different placing of elements.
