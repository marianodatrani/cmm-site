---
title: Merging spreadsheet or csv files into one dataframe - Part 3
author: Mariano
date: '2022-09-19'
slug: [merging-files-part-3]
categories:
  - File merging
tags:
  - rrrrrrrr
subtitle: ''
excerpt: 'A workflow to read data from separate files with a unified format, merge them into a single data frame, then export them as one file. Part 3 - dealing with untidy columns and rows.'
draft: yes
series: ~
layout: single
---

Benvenuti again. Today we improve further our workflow started in [Part 1](https://datamariano.netlify.app/blog/2022-09-07-merging-spreadsheet-or-csv-files-into-one-dataframe-part-1/) and expanded in [Part 2](https://datamariano.netlify.app/blog/2022-09-14-merging-spreadsheet-or-csv-files-into-one-dataframe-part-2/). In these previous episodes we used files with clean header-row-column format so the reading function could do its work with only filename provided.  \
As always, we start with loading the packages we will use, continue with setting the working directory, get all filenames we want to read with the `list.files()` function and print our `filenames` to check them. This time we use the `readxl` package instead of `readr`, because our files this time have ".xls" and ".xlsx" extensions.


```r
library(dplyr)
library(readxl)
library(tibble)
library(stringr)
library(lubridate)
```




```r
setwd("path/to/Downloads2020")
```


```r
filenames <- list.files(pattern = ".*xlsx$")
```


```r
filenames
```

```
##  [1] "StopTheWar20150121.xlsx" "StopTheWar20150224.xlsx"
##  [3] "StopTheWar20150318.xlsx" "StopTheWar20150417.xlsx"
##  [5] "StopTheWar20150505.xlsx" "StopTheWar20150528.xlsx"
##  [7] "StopTheWar20150707.xlsx" "StopTheWar20150724.xlsx"
##  [9] "StopTheWar20150813.xlsx" "StopTheWar20150827.xlsx"
## [11] "StopTheWar20150917.xlsx" "StopTheWar20151009.xlsx"
## [13] "StopTheWar20151106.xlsx" "StopTheWar20151208.xlsx"
## [15] "StopTheWar20160107.xlsx" "T-1000_20150108.xlsx"   
## [17] "T-1000_20150121.xlsx"    "T-1000_20150224.xlsx"   
## [19] "T-1000_20150318.xlsx"    "T-1000_20150417.xlsx"   
## [21] "T-1000_20150505.xlsx"    "T-1000_20150528.xlsx"   
## [23] "T-1000_20150707.xlsx"    "T-1000_20150724.xlsx"   
## [25] "T-1000_20150813.xlsx"    "T-1000_20150827.xlsx"   
## [27] "T-1000_20150917.xlsx"    "T-1000_20151009.xlsx"   
## [29] "T-1000_20151106.xlsx"    "T-1000_20151208.xlsx"   
## [31] "T-1000_20160107.xlsx"
```

We have two series of files check one of each. important to know that we have no information about files before read in the first file ![stw image](stopthewar.jpg) we have columns \
in the other ![t1000 image](t1000.jpg) we have more columns












