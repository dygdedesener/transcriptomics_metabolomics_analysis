## Introduction

In this script, visualization of the enriched pathway which include both
altered genes and metabolites is performed.

## Setup

``` r
# check if libraries are already installed > otherwise install it
if (!requireNamespace("BiocManager", quietly = TRUE)) install.packages("BiocManager")
if(!"rstudioapi" %in% installed.packages()) BiocManager::install("rstudioapi")
if(!"RCy3" %in% installed.packages()) BiocManager::install("RCy3")
if(!"rWikiPathways" %in% installed.packages()) BiocManager::install("rWikiPathways")
if(!"RColorBrewer" %in% installed.packages()) BiocManager::install("RColorBrewer")
if(!"dplyr" %in% installed.packages()) BiocManager::install("dplyr")
#load libraries
library(rstudioapi)
library(RCy3)#connect cytoscape via R
library(rWikiPathways)#to get pathways from WikiPathways
library(RColorBrewer)#to manage colors with R
library(dplyr)
# set your working environment to the location where your current source file is saved into.
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
```

## Import pathway

``` r
#make sure to launch cytoscape, if you get CyREST error you need to relaunch cytoscape
cytoscapePing()
#close all opened session before starting
closeSession(FALSE)
#pathway IDs to be visualized
pathway.id <- "WP4723"# omega-3/omega-6 fatty acid synthesis pathway is one of the enriched pathways in our study
#import pathways as pathway in cytoscape
RCy3::commandsRun(paste0('wikipathways import-as-pathway id=',pathway.id )) 
```

## Data upload

``` r
# to read data as input we need go back to previous branch
setwd('..')
combined.data <- read.delim("10-identifier_mapping/output/combinedData")
#set wd back to the main folder
setwd(dirname(rstudioapi::getSourceEditorContext()$path))

#get node table from imported pathway in cytoscape
ID.cols <- getTableColumns(table ="node", columns = c("XrefId","Ensembl", "ChEBI"))
#filter out rows which contain NA value for columns Ensembl and ChEBI
ID.cols <- ID.cols[!with(ID.cols, is.na(Ensembl) & is.na(ChEBI)),]
#if a row value in the Ensembl column is NA then replace it with ChEBI  
ID.cols$Ensembl <- ifelse(is.na(ID.cols$Ensembl), ID.cols$ChEBI, ID.cols$Ensembl)
#use the only one column contains both Ensembl and ChEBI identifiers
ID.cols <- data.frame(ID.cols[,c(1,2)])
#change column name
colnames(ID.cols)[2] <- "omics.ID"

#merge two data frames for adding xrefid to the combined data frame
data <- merge(combined.data, ID.cols, by.x = "Identifier", by.y = "omics.ID" )
#remove duplicate rows
data <- data %>% distinct(Identifier, .keep_all = TRUE)
#change column order
data <- data [,c(5,1,2,3,4)]
colnames(data)[2] <- "omics.ID"
#load data to the imported pathway in cytoscape by key column as XrefId
loadTableData(table = "node", data = data, data.key.column = "XrefId", table.key.column = "XrefId")
```

    ## [1] "Success: Data loaded in defaultnode table"

## Visualization options

``` r
#new visual style is created
RCy3::copyVisualStyle("default","pathwayStyle")
#set new style as the current style
RCy3::setVisualStyle("pathwayStyle")
```

    ##                 message 
    ## "Visual Style applied."

``` r
#set node dimensions as fixed sizes
RCy3::lockNodeDimensions(TRUE, style.name="pathwayStyle")

#node shape mapping
RCy3::setNodeShapeMapping('Type',c('GeneProduct','Protein', 'Metabolite'),c('ELLIPSE','ELLIPSE','RECTANGLE'),style.name="pathwayStyle")
```

    ## NULL

``` r
#change node height
RCy3::setNodeHeightMapping('Type',c('GeneProduct','Protein', 'Metabolite'), c(23,23,25), mapping.type = "d", style.name = "pathwayStyle")
```

    ## NULL

``` r
#change node width
RCy3::setNodeWidthMapping('Type',c('GeneProduct','Protein', 'Metabolite'), c(60,60,175), mapping.type = "d", style.name = "pathwayStyle")
```

    ## NULL

``` r
#set node color based on log2FC for both genes and metabolites
node.colors <- c(rev(brewer.pal(3, "RdBu")))
setNodeColorMapping("log2FC", c(-1,0,1), node.colors, default.color = "#D3D3D3", style.name = "pathwayStyle")
```

    ## NULL

``` r
#set node border width and color based on p-value
#first we need to get all p-values from node table
pvalues = getTableColumns(table = 'node',columns = c('name','pvalue'))  
min.pval = min(pvalues[,2],na.rm=TRUE)
#get non-significant nodes (p-value>0.05)
nonsign.nodes <- pvalues  %>% filter(pvalue > 0.05, na.rm = TRUE)

#set node border width for all nodes which has a p-value
setNodeBorderWidthMapping('pvalue', c(min.pval,0.05), c(10,10) , mapping.type = "c", style.name = "pathwayStyle")
```

    ## NULL

``` r
#set node border color for all nodes which has a p-value
setNodeBorderColorMapping('pvalue', c(min.pval,0.05), c('#00FF00','#00FF00'), style.name = "pathwayStyle")
```

    ## NULL

``` r
#filter our non-sign nodes from the visualization
RCy3::setNodeBorderWidthBypass(nonsign.nodes$name, new.sizes = 2)
RCy3::setNodeBorderColorBypass(nonsign.nodes$name, new.colors = "#D3D3D3")

#save output 
png.file <- file.path(getwd(), "output/omics_visualization.png")
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                                                                                      file 
    ## "C:\\Users\\dedePC\\Desktop\\FNS_CLOUD\\GITHUB_codes\\Transcriptomics_Metabolomics_Analysis\\visualization_multiomics\\11-visualization\\output\\omics_visualization.png"

``` r
#Jupyter Notebook file
if(!"devtools" %in% installed.packages()) BiocManager::install("devtools")
devtools::install_github("mkearney/rmd2jupyter", force=TRUE)
```

    ## 
    ## * checking for file 'C:\Users\dedePC\AppData\Local\Temp\RtmpwrGTN8\remotes3a3ce9d69ff\mkearney-rmd2jupyter-d2bd2aa/DESCRIPTION' ... OK
    ## * preparing 'rmd2jupyter':
    ## * checking DESCRIPTION meta-information ... OK
    ## * checking for LF line-endings in source and make files and shell scripts
    ## * checking for empty or unneeded directories
    ## Omitted 'LazyData' from DESCRIPTION
    ## * building 'rmd2jupyter_0.1.0.tar.gz'
    ## 

``` r
library(devtools)
library(rmd2jupyter)
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
rmd2jupyter("visualization.Rmd")
```