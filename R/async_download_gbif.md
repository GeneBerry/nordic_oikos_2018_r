Asyncronous download using the rgibf package
================
AGF
November 15, 2017

Asyncronous download using the rgibf package
============================================

Procedures and examples on downloading us ing registred downloads (as opposite to the download without registration through e.g. rgbif). This has several adwantages. The most prominent are:

-   You are not restricted to a hard limit of 200 000 records
-   You get a citation of your downloaded data that can/must be used when citing your data usage in e.g. a scientific publication or thesis.

procedure
---------

The following chunks of script do the following a) load required packages b) perform a simple search for getting example data, including find species of interest, construct query and finally request a download key from GBIF c) Download and save data d) Create data.frame from the downloaded data

Load packages
-------------

``` r
library(rgbif)
library(stringr)
library(rio)
library(dplyr)
```

Search and request download key
===============================

This is done in three steps. First we find a species key (for convinience, we do this step outside the request for download key) using the name\_suggest function. Secondly, we request the download key by sending an API call to GBIF using the occ\_download command. Then we request the download. For convinience, we here add a little [time-delay function](https://gist.githubusercontent.com/andersfi/1e7cd54cf4d12e86f0ecc66effd86129/raw/0d40d1971427aecd0c469c062c0693320392435b/download_from_GBIF_key) so that we don't have to watch for e-mails arriving from GBIF, but instead can go and have a cup of coffee. Finally, download the data.

The current script is set up to store the downloaded data to your disk (in your R working directory). This is convinient if you work with large datasets and want to store the data for re-use in later sessions. You can alternatively download to a temporary file.

``` r
# 1. we find a taxon key - get list of gbif key's to filter download
key <- name_suggest(q='Esox lucius', rank='species')$key[1] 
#key2 <- name_suggest(q='Actinopterygii', rank='class')$key[1] 
#key3 <- name_suggest(q='Carassius carassius', rank='species')$key[1]

# paste("https://www.gbif.org/species/",key3,sep="") gives you the homepage of the species

# Get callback key from GBIF API and construct download url. 
# set user_name, e-mail, and pswd as global options first
# NB: This is set up to request userdetails interactivly, modify if not running through rstudio
options(gbif_user=rstudioapi::askForPassword("my gbif username"))
options(gbif_email=rstudioapi::askForPassword("my registred gbif e-mail"))
options(gbif_pwd=rstudioapi::askForPassword("my gbif password"))

# Get download key. NB! Maximum of 3 download request handled simultaniusly
download_key <- occ_download('taxonKey = 2346633,2366645','hasCoordinate = TRUE',
                             'hasGeospatialIssue = FALSE',
                             'country = NO',
                             #'geometry = POLYGON((9.33 62.80,9.33 64.20,12.13,64.20,12.13,62.80,9.33 62.80))',
                             type="and") %>% 
  occ_download_meta

# Automatize the process: Script for calling the download at regular interval
# - download_key from occ_download, n_try=number of trials before giving up, 
# Sys.sleep_duration=time in seconds between each trial (adjust after the expected size of the download). This function will download the data as a .zip file to the working directory of R. 

source("https://gist.githubusercontent.com/andersfi/1e7cd54cf4d12e86f0ecc66effd86129/raw/0d40d1971427aecd0c469c062c0693320392435b/download_from_GBIF_key")
download_GBIF_API(download_key=download_key,n_try=5,Sys.sleep_duration=15)

# Alternatively, wait for e-mail or watch on GBIF portal: https://www.gbif.org/user/download # The download key will be shown as lasts part of the url e.g. https://www.gbif.org/occurrence/download/0003580-171002173027117
download.file(url=paste("http://api.gbif.org/v1/occurrence/download/request/",
                        download_key[1],sep=""),
              destfile=paste(download_key[1],".zip",sep=""),
              quiet=FALSE)
```

Open the data and extract into data.frame
=========================================

The download gives us back a package with data and metadata boundled togheter in a .zip file. This includes both the metadata, citations of the orginal datasets that the occurrence download is a composit of, the licences, as well as the data in both gbif interpreted form (occurrence.txt) and the raw data as provided by the user (verbatim.txt). It is usually the interpreted data you want to use (occurrence.txt).

``` r
# Get a list of the files within the archive by using "list=TRUE" in the unzip function.
unzip(paste(download_key[1],".zip",sep=""),list=T)

# Get the occurrence.txt file in as a dataframe (using import from rio)
occurrence <- import(unzip(zipfile=paste(download_key[1],".zip",sep=""),
                 files="occurrence.txt"),header=T,sep="\t")

# Finally, but nimportant - Citation 
paste("GBIF Occurrence Download", download_key[2], "accessed via GBIF.org on", Sys.Date())
```