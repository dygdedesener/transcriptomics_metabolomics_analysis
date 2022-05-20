## Introduction

In this workflow, we will create protein-protein interaction networks
for both biopsy locations ileum and rectum Then these networks will be
extended with pathways from WikiPathways database to create
protein-protein-pathway interaction networks. Finally, MCL (Markov
Clustering) network clustering algorithm will be applied for both
networks.

## Setup

Installing and loading required libraries

## PPI network for rectum biopsy location

``` r
#we will read each input data files 
#read input data
deg <- read.csv("data/DEG.overlapped_rectum", sep = "\t")
#filter out genes that does not have ENTREZ ID 
deg <- deg %>% tidyr:: drop_na(ENTREZ)
#check that cytoscape is connected
cytoscapePing()
#create a PPI network using overlapped DEGs between CD and UC, the creation will be done 
x <- readr::format_csv(as.data.frame(deg$ENTREZ), col_names=F, escape = "double", eol =",")
commandsRun(paste0('string protein query cutoff=0.7 newNetName="PPI_network_rectum" query="',x,'" limit=0'))
```

    ## [1] "Loaded network 'STRING network - PPI_network_rectum' with 1027 nodes and 2534 edges"

``` r
# this function converts a command string into a CyREST query URL, executes a GET request, 
#and parses the result content into an R list object. Same as commandsGET
#get proteins (nodes) from the constructed network
proteins <- RCy3::getTableColumns(columns=c("query term", "display name"))
#get edges from the network
ppis     <- RCy3::getTableColumns(table="edge", columns=c("name"))
#split extracted edge information into source-target format
ppis     <- data.frame(do.call('rbind', strsplit(as.character(ppis$name),' (pp) ',fixed=TRUE)))
#merge obtained nodes and edges to get entrez IDs for each source genes 
ppis.2   <- merge(ppis, proteins, by.x="X1", by.y="display name", all.x=T)
#change column names
colnames(ppis.2) <- c("s", "t", "source")
#merge again to add entrez IDs of target genes 
ppis.3   <- merge(ppis.2, proteins, by.x="t", by.y="display name", all.x=T)
colnames(ppis.3)[4] <-"target"
#ppi3 stores interaction between all proteins so add new column represeting type of interaction
ppis.3$interaction <- "PPI"
#add col names to protein
colnames(proteins) <- c("id","label")
proteins$type <- "protein"

###############get all pathways from WIKIPATHWAYS #################
#below code should be performed first to handle the ssl certificate error uploading pathways 
options(RCurlOptions = list(cainfo = paste0( tempdir() , "/cacert.pem" ), ssl.verifypeer = FALSE))
wp.hs.gmt <- rWikiPathways::downloadPathwayArchive(organism="Homo sapiens", format = "gmt")
#Now that we have the latest GMT file for human pathways, 
#all wp and gene information stored in wp2gene object
wp2gene   <- rWikiPathways::readPathwayGMT(wp.hs.gmt)
#filter out  pathways that does not consist of any differentially expressed genes 
wp2gene.filtered <- wp2gene [wp2gene$gene %in% deg$ENTREZ,]

#change column names 
colnames(wp2gene.filtered)[3] <- c("source")
colnames(wp2gene.filtered)[5] <- c("target")
#add new column for representing interaction type
wp2gene.filtered$interaction <- "Pathway-Gene"

#store only wp information 
pwy.filtered <- unique( wp2gene [wp2gene$gene %in% deg$ENTREZ,c(1,3)])
colnames(pwy.filtered) <- c("label", "id")
pwy.filtered$type <- "pathway"
colnames(pwy.filtered) <- c("label","id", "type")

#get genes 
genes <- unique(deg[,c(1,2)])
genes$type <- "gene"
colnames(genes) <- c("id","label","type")
genes$id <- as.character(genes$id)
#genes and pathways are separate nodes and they need to be merged
nodes.ppi <- dplyr::bind_rows(genes,pwy.filtered)
rownames(nodes.ppi) <- NULL
edges.ppi <- unique(dplyr::bind_rows(ppis.3[,c(3,4,5)], wp2gene.filtered[,c(3,5,6)]))
rownames(edges.ppi) <- NULL
```

## Create PPI-pathway network for rectum biopsy location

``` r
###########Create PPI-pathway network###
RCy3::createNetworkFromDataFrames(nodes= nodes.ppi, edges = edges.ppi, title="PPI_Pathway_Network_rectum", collection="rectum")
```

    ## networkSUID 
    ##      721489

``` r
RCy3::loadTableData(nodes.ppi, data.key.column = "label", table="node", table.key.column = "label")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
RCy3::loadTableData(deg, data.key.column = "ENTREZ", table.key.column = "id")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
###########Visual style#################
RCy3::copyVisualStyle("default","ppi")#Create a new visual style (ppi) by copying a specified style (default)
RCy3::setNodeLabelMapping("label", style.name="ppi")
```

    ## NULL

``` r
RCy3::lockNodeDimensions(TRUE, style.name="ppi")#Set a boolean value to have node width and height fixed to a single size value.
#threshold is set based of differentiall expressed gene criteria
data.values<-c(-0.58,0,0.58) 
#red-blue color schema chosen
node.colors <- c(brewer.pal(length(data.values), "RdBu"))
#nodes are splitted to shown both log2fc values on both biospy location 
RCy3::setNodeCustomHeatMapChart(c("log2FC_CD","log2FC_UC"), slot = 2, style.name = "ppi", colors = node.colors)

#Set the bypass value for fill color for the specified node or nodes.
selected <- RCy3::selectNodes(nodes="pathway", by.col = "type")
RCy3::setNodeShapeBypass(node.names = selected$nodes, new.shapes = "hexagon")
RCy3::setNodeColorBypass(node.names = selected$nodes, "#FFFFCE")
RCy3::setNodeBorderColorBypass(node.names = selected$nodes, "#000000")
RCy3::setNodeBorderWidthBypass(node.names = selected$nodes, 4)
RCy3::setVisualStyle("ppi")
```

    ##                 message 
    ## "Visual Style applied."

``` r
RCy3::toggleGraphicsDetails()

# Saving output
if(dir.exists("output"))#if the output folder already exist
  unlink("output", recursive=TRUE)#firs delete the existing one
dir.create("output")#create a new output folder
png.file <- file.path(getwd(), "output/PPI_Pathway_Network_rectum.png")
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                      file 
    ## "C:\\Users\\dedePC\\Desktop\\FNS_CLOUD\\Network_analysis\\GITHUB\\output\\PPI_Pathway_Network_rectum.png"

## PPI network for ileum biopsy location

``` r
#clear environment
rm(list=ls())

#read ileum DEG data
deg <- read.csv("data/DEG.overlapped_ileum", sep = "\t")
#filter out genes that does not have ENTREZ ID 
deg <- deg %>% tidyr:: drop_na(ENTREZ)
#check that cytoscape is connected
cytoscapePing()

#create a PPI network for ileum location
x <- readr::format_csv(as.data.frame(deg$ENTREZ), col_names=F, escape = "double", eol =",")
commandsRun(paste0('string protein query cutoff=0.7 newNetName="PPI_network_ileum" query="',x,'" limit=0'))
```

    ## [1] "Loaded network 'STRING network - PPI_network_ileum' with 1026 nodes and 2533 edges"

``` r
#get proteins (nodes) from the constructed network
proteins <- RCy3::getTableColumns(columns=c("query term", "display name"))
#get edges from the network
ppis     <- RCy3::getTableColumns(table="edge", columns=c("name"))
#split extracted edge information into source-target format
ppis     <- data.frame(do.call('rbind', strsplit(as.character(ppis$name),' (pp) ',fixed=TRUE)))
#merge obtained nodes and edges to get entrez IDs for each source genes 
ppis.2   <- merge(ppis, proteins, by.x="X1", by.y="display name", all.x=T)
#change column names
colnames(ppis.2) <- c("s", "t", "source")
#merge again to add entrez IDs of target genes 
ppis.3   <- merge(ppis.2, proteins, by.x="t", by.y="display name", all.x=T)
colnames(ppis.3)[4] <-"target"

#ppi3 stores interaction between all proteins so add new column represeting type of interaction
ppis.3$interaction <- "PPI"
#add col names to protein
colnames(proteins) <- c("id","label")
proteins$type <- "protein"

###############get all pathways from WIKIPATHWAYS #################
#below code should be performed first to handle the ssl certificate error uploading pathways 
options(RCurlOptions = list(cainfo = paste0( tempdir() , "/cacert.pem" ), ssl.verifypeer = FALSE))
wp.hs.gmt <- rWikiPathways::downloadPathwayArchive(organism="Homo sapiens", format = "gmt")
#Now that we have the latest GMT file for human pathways, 
#all wp and gene information stored in wp2gene object
wp2gene   <- rWikiPathways::readPathwayGMT(wp.hs.gmt)
#filter out  pathways that does not consist of any differentially expressed genes 
wp2gene.filtered <- wp2gene [wp2gene$gene %in% deg$ENTREZ,]

#change column names 
colnames(wp2gene.filtered)[3] <- c("source")
colnames(wp2gene.filtered)[5] <- c("target")
#add new column for representing interaction type
wp2gene.filtered$interaction <- "Pathway-Gene"

#store only wp information 
pwy.filtered <- unique( wp2gene [wp2gene$gene %in% deg$ENTREZ,c(1,3)])
colnames(pwy.filtered) <- c("label", "id")
pwy.filtered$type <- "pathway"
colnames(pwy.filtered) <- c("label","id", "type")

#get genes 
genes <- unique(deg[,c(1,2)])
genes$type <- "gene"
colnames(genes) <- c("id","label","type")
genes$id <- as.character(genes$id)
#genes and pathways are separate nodes and they need to be merged
nodes.ppi <- dplyr::bind_rows(genes,pwy.filtered)
rownames(nodes.ppi) <- NULL
edges.ppi <- unique(dplyr::bind_rows(ppis.3[,c(3,4,5)], wp2gene.filtered[,c(3,5,6)]))
rownames(edges.ppi) <- NULL
```

## Create PPI-pathway network for ileum biopsy location

``` r
####Create PPI-pathway network###
RCy3::createNetworkFromDataFrames(nodes= nodes.ppi, edges = edges.ppi, title="PPI_Pathway_Network_ileum",collection="ileum")
```

    ## networkSUID 
    ##      777799

``` r
RCy3::loadTableData(nodes.ppi, data.key.column = "label", table="node", table.key.column = "label")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
RCy3::loadTableData(deg, data.key.column = "ENTREZ", table.key.column = "id")
```

    ## [1] "Success: Data loaded in defaultnode table"

``` r
###########Visual style########
RCy3::copyVisualStyle("default","ppi")
RCy3::setNodeLabelMapping("label", style.name="ppi")
```

    ## NULL

``` r
RCy3::lockNodeDimensions(TRUE, style.name="ppi")
RCy3::setNodeShapeMapping('type', c('gene','pathway'), c("ellipse","rectangle"), style.name="ppi")
```

    ## NULL

``` r
RCy3::setNodeSizeMapping('type', c('gene','pathway'), c(40,25), mapping.type = "d", style.name = "ppi")
```

    ## NULL

``` r
data.values<-c(-0.58,0,0.58) #try like this 
node.colors <- c(rev(brewer.pal(length(data.values), "RdBu")))
RCy3::setNodeCustomHeatMapChart(c("log2FC_CD","log2FC_UC"), slot = 2, style.name = "ppi", colors = c("#CC3300","#FFFFFF","#6699FF","#CCCCCC"))          
RCy3::setVisualStyle("ppi")
```

    ##                 message 
    ## "Visual Style applied."

``` r
RCy3::toggleGraphicsDetails()

# Saving output
png.file <- file.path(getwd(), "output/PPI_Pathway_Network_ileum.png")
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                     file 
    ## "C:\\Users\\dedePC\\Desktop\\FNS_CLOUD\\Network_analysis\\GITHUB\\output\\PPI_Pathway_Network_ileum.png"

``` r
# save cytoscape session 
cys.file <- file.path(getwd(), "output/PPI_Pathway_Network_rectum_ileum.cys")
saveSession(cys.file) 
```

## Clustering obtained networks

``` r
#we will continue with the same session used for ppi-pathway networks
#to check cytoscape is connected
cytoscapePing()

#to get network name of the rectum 
networkSuid = getNetworkSuid("PPI_Pathway_Network_rectum")
setCurrentNetwork(network=networkSuid)
#create cluster command
clustermaker <- paste("cluster mcl createGroups=TRUE showUI=TRUE network=SUID:",networkSuid, sep="")
#run the command in cytoscape
res <- commandsGET(clustermaker)
#total number of clusters for rectum
num_rectum <- as.numeric(gsub("Clusters: ", "", res[1]))
#export image
png.file <- file.path(getwd(), "output/PPI_Pathway_Network_rectum--clustered.png")
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                                 file 
    ## "C:\\Users\\dedePC\\Desktop\\FNS_CLOUD\\Network_analysis\\GITHUB\\output\\PPI_Pathway_Network_rectum--clustered.png"

``` r
#to get network name of the ileum 
networkSuid = getNetworkSuid("PPI_Pathway_Network_ileum")
setCurrentNetwork(network=networkSuid)
#create cluster command
clustermaker <- paste("cluster mcl createGroups=TRUE showUI=TRUE network=SUID:",networkSuid, sep="")
#run the command in cytoscape
res <- commandsGET(clustermaker)
#total number of clusters for rectum
num_ileum <- as.numeric(gsub("Clusters: ", "", res[1]))
#export image
png.file <- file.path(getwd(), "output/PPI_Pathway_Network_ileum--clustered.png")
exportImage(png.file,'PNG', zoom = 500)
```

    ##                                                                                                                file 
    ## "C:\\Users\\dedePC\\Desktop\\FNS_CLOUD\\Network_analysis\\GITHUB\\output\\PPI_Pathway_Network_ileum--clustered.png"

``` r
#save the session
cys.file <- file.path(getwd(), "output/PPI_Pathway_Network_clustered_rectum_ileum.cys")
saveSession(cys.file) 
```