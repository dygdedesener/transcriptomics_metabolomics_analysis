## Introduction

In this script, filtering options will be applied for transcriptomics
data to be prepared for the analysis

## Setup

``` r
# check if libraries are already installed > otherwise install it
if(!requireNamespace("BiocManager", quietly = TRUE)) install.packages("BiocManager",repos = "http://cran.us.r-project.org")
if(!"rstudioapi" %in% installed.packages()) BiocManager::install("rstudioapi")
if(!"readxl" %in% installed.packages()) BiocManager::install("readxl")
if(!"dplyr" %in% installed.packages()) BiocManager::install("dplyr")

library(rstudioapi)
library(readxl)
library(dplyr)

# set your working environment to the location where your current source file is saved into.
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
#you can check where your current working directory is
#getwd()
```

## Read and filter out metadata

``` r
#we first read metadata
htxMeta <- read.csv("hmp2_metadata_2018-08-20.csv")
#filter out by data type as host-transcriptomics
htxMeta <- htxMeta  %>% filter(htxMeta$data_type == "host_transcriptomics")

#filter out data by biopsy location, we take CD, UC and nonIBD samples on ileum and  rectum 
htxMeta <-htxMeta  %>% filter(  (htxMeta$diagnosis == "CD" & htxMeta$biopsy_location=="Ileum") 
                                |  (htxMeta$diagnosis == "CD" & htxMeta$biopsy_location=="Rectum")
                                |  (htxMeta$diagnosis == "UC" & htxMeta$biopsy_location=="Ileum") 
                                |  (htxMeta$diagnosis == "UC" & htxMeta$biopsy_location=="Rectum") 
                                |  (htxMeta$diagnosis == "nonIBD" & htxMeta$biopsy_location=="Rectum") 
                                |  (htxMeta$diagnosis == "nonIBD" & htxMeta$biopsy_location=="Ileum") 
)
#filter out samples by visit_num=1
htxMeta <-htxMeta  %>% filter(htxMeta$visit_num == "1")

#filter out unused columns
htxMeta <- htxMeta %>% select(External.ID,Participant.ID,biopsy_location,diagnosis )

#we ordered htxMeta data based on external ID to match samples with htx count correctly
htxMeta<- htxMeta[order(htxMeta$External.ID),]#order htxMeta by external ID
```

# Filter out host transcriptomics (htx) count data based on sample names obtained from htx meta data

``` r
#transcript count (htx count) original file is read
htxOrj <- read.csv("host_tx_counts.tsv",sep = "\t")

#we should convert sample names to upper because some of them are in lower case
colnames(htxOrj)<-toupper(colnames(htxOrj))

#htx count data are filtered based on col names in htxMeta
names.use <- names(htxOrj)[(names(htxOrj) %in% htxMeta$External.ID)]
#filter out htxOrj based on names.use and create htxCount
htxCount <- htxOrj[, names.use]
#htxCount data are ordered based on column names to match samples between htxCount and sampleLabels
htxCount <- htxCount[,order(names(htxCount))]
#check whether they are in same order
#colnames(htxCount) == htxMeta[,"External.ID"]

#finally we write all the generated data into the related output files 
write.table(htxCount, "output/htxCount", sep="\t",quote=FALSE, row.names = TRUE )
write.table(htxMeta, "output/metaData", sep="\t",quote=FALSE,row.names = FALSE, col.names = FALSE)
```