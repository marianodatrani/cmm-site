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
draft: false
series: ~
layout: single
---

Benvenuti again. Today we improve further our workflow started in [Part 1](https://datamariano.netlify.app/blog/2022-09-07-merging-spreadsheet-or-csv-files-into-one-dataframe-part-1/) and expanded in [Part 2](https://datamariano.netlify.app/blog/2022-09-14-merging-spreadsheet-or-csv-files-into-one-dataframe-part-2/). In these earlier episodes we used files with clean header-row-column format so the reading function could do its work with only filename provided. This time we have to deal with dirty file headers and unneeded columns, meaning that in order to read the data properly, these issues must be solved. \
\
As always, we start with loading the packages we will use, continue with setting the working directory, get all filenames we want to read with the `list.files()` function and print our `filenames` to check them. In this episode we use the `readxl` package instead of `readr`, because our files have ".xlsx" extension.


```r
library(dplyr)
library(readxl)
library(tibble)
library(stringr)
library(lubridate)
```




```r
setwd("path/to/Downloads2015")
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
## [23] "T-1000_20150702.xlsx"    "T-1000_20150707.xlsx"   
## [25] "T-1000_20150723.xlsx"    "T-1000_20150724.xlsx"   
## [27] "T-1000_20150812.xlsx"    "T-1000_20150813.xlsx"   
## [29] "T-1000_20150826.xlsx"    "T-1000_20150827.xlsx"   
## [31] "T-1000_20150917.xlsx"    "T-1000_20151009.xlsx"   
## [33] "T-1000_20151106.xlsx"    "T-1000_20151208.xlsx"   
## [35] "T-1000_20160107.xlsx"
```

We have two series of files with device names "StopTheWar" and "T-1000" and we glance at one of each in Excel to get some information before reading. \
\
![stw image](stopthewar.jpg) \
\
In the "StopTheWar" files there are 9 columns with deliberately idiot and long names, similarly to those I had to deal with in my work. Some columns seem to have only the name because values occur only sporadically in them. There is also a title in the very first row so column names are in the second. Reading these files without specifying the first row to skip would result in a `data.frame` with the first column named after the title and the rest like "...2" and "...3" as the function couldn't find more names in the first row where column names should be by default. Additionally, every column would have character type despite we have a datetime and some numeric columns, because this way our original column names are considered as text values, forcing the the whole column into character type, since data type of values in the same column cannot differ, as we'll see. Let's check the other type of files.\
\
![t1000 image](t1000.jpg) \
\
In the "T-1000" files we have two more columns filled with numeric values, because they are from a different kind of device. Note that in both types the name of these numeric columns starts with "Unit", followed by "X" in "StopTheWar" and "Y" in "T-1000", referring to the unit of measurement. These are fake names (surprisingly, in these fake files), but based on real ones, and the point is that the columns have standard naming with only small differences. And we are going to use these differences in our reading loop to distinguish between the two types of files. \
\

As in the previous parts we continue with defining objects needed for our loop. First we create and print the `logger_names` vector. The regex pattern in `str_replace()` was explained in [Part 2](https://datamariano.netlify.app/blog/2022-09-14-merging-spreadsheet-or-csv-files-into-one-dataframe-part-2/).


```r
logger_names <- unique(str_replace(filenames, "([^_]+)(?:_)*\\d{8}.+", "\\1"))
```


```r
logger_names
```

```
## [1] "StopTheWar" "T-1000"
```

Then we get the `actual_year`, using this vector in `make_datetime()` functions when creating the one column `real_time` for the whole calendar year. In these files we have 10 minute time intervals so we set the "by" argument representing the desired time step of `seq.POSIXt()` to 600 seconds.




```r
actual_year <- as.integer(str_extract(getwd(), "\\d{4}"))
```


```r
real_time <- as_tibble_col(seq.POSIXt(make_datetime(actual_year, 1, 1, 0, 0, 0), 
                                  make_datetime(actual_year, 12, 31, 23, 55, 0), 600), 
                       column_name = "Datetime")
```

After these now familiar steps, let's check what happens if we read a file without defining further arguments in `read_xlsx()`.


```r
a_t_1000_file <- read_xlsx("T-1000_20150318.xlsx")
```

```
## New names:
## * `` -> ...2
## * `` -> ...3
## * `` -> ...4
## * `` -> ...5
## * `` -> ...6
## * ...
```

```r
head(a_t_1000_file)
```

```
## # A tibble: 6 x 11
##   `Title: T-1000__~` ...2  ...3  ...4  ...5  ...6  ...7  ...8  ...9  ...10 ...11
##   <chr>              <chr> <chr> <chr> <chr> <chr> <chr> <chr> <chr> <chr> <chr>
## 1 #                  Date~ Unit~ Unit~ Unit~ Unit~ Some~ Anot~ Othe~ Acti~ Cont~
## 2 1                  02.2~ -16.~ -11.~ 1.73~ 27.8~ <NA>  <NA>  <NA>  <NA>  <NA> 
## 3 3                  02.2~ 6.38~ 83.4~ 57.4~ 29.1~ <NA>  <NA>  <NA>  <NA>  <NA> 
## 4 5                  02.2~ 5.60~ 87.5~ -19.~ 80.9~ <NA>  <NA>  <NA>  <NA>  <NA> 
## 5 7                  02.2~ 70.4~ -17.~ 94.2~ 44.1~ <NA>  <NA>  <NA>  <NA>  <NA> 
## 6 9                  02.2~ 82.5~ 0.02~ 61.7~ 38.4~ <NA>  <NA>  <NA>  <NA>  <NA>
```

So we have these above mentioned artificial new column names, indicated already in the function message, and character data types. We have to get rid of our first row and thus enable the function to recognize the real column names and data types. In addition to that, we have some superfluous columns like the first one with measurement number, which too can be excluded in this step. In fact, only the datetime and columns whose name starts with "Unit" are needed.\
\
\
Loop. \
In this occasion we introduce the pipe "%>%" from the `magrittr` package (loaded with `dplyr`) into the loop to keep our code shorter. The behavior of this pipe can be explained like it considers what is before the pipe as the first argument of the following function (after the "%>%"). It means that functions in a pipeline miss their first argument, typically a `data.frame` to operate on, making these longer scripts more readable and less error-prone. \
\
As previously, we define the `files_for_one_logger`, followed by initiating an empty `list_for_dataframes`, then an inner loop over `files_for_one_logger` to read the files into the list as `data.frame`s and binding their rows into a `combined_df`, then `left_join()` it with `real_time` to get our `final_df` that we `assign()` into an individual `data.frame` object using the `logger_names` and `actual_year` vectors. \
\
To solve the described header-column problems we set the "skip" argument to 1 in `read_xlsx()` to omit the first row and start reading from the second, then pipe a few cleaning steps after this function. In this case our columns can be identified by the first part of their name, so we give the appropriate strings "Date" and "Unit" to the helper function `starts_with()` inside `select()` to keep only columns needed. With `rename()`we give its proper name to Datetime, choosing the column again with `starts_with()`. For the other columns we give suitable names with `rename_at()`, applying `starts_with()` once more, nested inside `vars()`, then `str_replace()` inside `funs()` to replace the whole column name with its relevant part. After that we use `mdy_hms()` from `lubridate` inside a `mutate()` function to convert our Datetime column into "POSIXct" and "POSIXt", and `mutate_if()` to turn the remaining character type columns into numeric. \
\
Because of the automatic naming of `read_xlsx()`, there should be a lot of warnings after running this loop, but we disabled warnings for this case to avoid getting overwhelmed by these messages.


```r
for (name in seq_along(logger_names)){ 
  
  files_for_one_logger <- list.files(pattern = str_c("^", logger_names[name], ".*.xlsx"))

  list_for_dataframes <- list()

  for (i in seq_along(files_for_one_logger)) {
    list_for_dataframes[[i]] <- read_xlsx(files_for_one_logger[i], skip = 1) %>% 
  select(starts_with(c("Date", "Unit"))) %>% 
  rename(Datetime=starts_with("Date")) %>% 
  rename_at(vars(starts_with("Unit")), funs(str_replace(., ".*(Sensor_\\d)", "\\1"))) %>%
  mutate(Datetime=mdy_hms(Datetime)) %>% 
  mutate_if(is.character, as.numeric)
  }

  combined_df <- bind_rows(list_for_dataframes)

  final_df <- left_join(real_time, combined_df, by="Datetime")
  
  assign(str_c(logger_names[name], "_", actual_year), final_df)

 rm(combined_df, final_df, i, name, files_for_one_logger)
 
}
```

Finito. \
To be continued with datetime issues in Part 4. \
\
Ci vediamo!
