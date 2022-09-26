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

Benvenuti again. Today we improve further our workflow started in [Part 1](https://datamariano.netlify.app/blog/2022-09-07-merging-spreadsheet-or-csv-files-into-one-dataframe-part-1/) and expanded in [Part 2](https://datamariano.netlify.app/blog/2022-09-14-merging-spreadsheet-or-csv-files-into-one-dataframe-part-2/). In these previous episodes we used files with clean header-row-column format so the reading function could do its work with only filename provided. This time we have to deal with dirty file headers and unneeded columns, meaning that in order to read the data properly, these issues must be solved. \
\
As always, we start with loading the packages we will use, continue with setting the working directory, get all filenames we want to read with the `list.files()` function and print our `filenames` to check them. In this episode we use the `readxl` package instead of `readr`, because our files have ".xls" and ".xlsx" extensions.


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
##  [1] "~$StopTheWar20150917.xlsx" "StopTheWar20150121.xlsx"  
##  [3] "StopTheWar20150224.xlsx"   "StopTheWar20150318.xlsx"  
##  [5] "StopTheWar20150417.xlsx"   "StopTheWar20150505.xlsx"  
##  [7] "StopTheWar20150528.xlsx"   "StopTheWar20150707.xlsx"  
##  [9] "StopTheWar20150724.xlsx"   "StopTheWar20150813.xlsx"  
## [11] "StopTheWar20150827.xlsx"   "StopTheWar20150917.xlsx"  
## [13] "StopTheWar20151009.xlsx"   "StopTheWar20151106.xlsx"  
## [15] "StopTheWar20151208.xlsx"   "StopTheWar20160107.xlsx"  
## [17] "T-1000_20150108.xlsx"      "T-1000_20150121.xlsx"     
## [19] "T-1000_20150224.xlsx"      "T-1000_20150318.xlsx"     
## [21] "T-1000_20150417.xlsx"      "T-1000_20150505.xlsx"     
## [23] "T-1000_20150528.xlsx"      "T-1000_20150707.xlsx"     
## [25] "T-1000_20150724.xlsx"      "T-1000_20150813.xlsx"     
## [27] "T-1000_20150827.xlsx"      "T-1000_20150917.xlsx"     
## [29] "T-1000_20151009.xlsx"      "T-1000_20151106.xlsx"     
## [31] "T-1000_20151208.xlsx"      "T-1000_20160107.xlsx"
```

We have two series of files with device names "StopTheWar" and "T-1000" and we glance at one of each in Excel to get some information before reading. \
![stw image](stopthewar.jpg) \
\
In the "StopTheWar" files there are 9 columns with deliberately idiot and long names, similarly to those I had to deal with in my work. Some columns seem to have only the name because values occur only sporadically in them. There is also a title in the very first row so column names are in the second. Reading these files without specifying the first row to skip would result in a `data.frame` with the first column named after the title and the rest like "...2" and "...3" as the function couldn't find more names in the first row where column names should be by default. Additionally, every column would have character type despite we have a datetime and some numeric columns, because this way our original column names are considered as text values, forcing the the whole column into character type, since data type of values in the same column cannot differ. Let's check the other type of files.\
\
![t1000 image](t1000.jpg) \
\
In the "T-1000" files we have two more columns filled with numeric values, because they are from a different kind of device. Note that in both types the name of these numeric columns starts with "Unit", followed by "X" in "StopTheWar" and "Y" in "T-1000", referring to the unit of measurement. These are artificial names we gave them, but the point is that the columns have standard naming with only small differences. And we are going to use these differences in our reading loop to distinguish between the two types of files.












