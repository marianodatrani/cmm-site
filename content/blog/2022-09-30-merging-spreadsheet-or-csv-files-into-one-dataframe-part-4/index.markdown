---
title: Merging spreadsheet or csv files into one dataframe - Part 4
author: Mariano
date: '2022-09-30'
slug: [merging-files-part-4]
categories:
  - File merging
tags:
  - rrrrrrrr
subtitle: ''
excerpt: 'A workflow to read data from separate files with a unified format, merge them into a single data frame, then export them as one file. Part 4 - dealing with datetime.'
draft: yes
series: ~
layout: single
---

Ciao! Working with date and time can be tricky so in this part we extend our workflow from [Part 1](https://datamariano.netlify.app/blog/2022-09-07-merging-spreadsheet-or-csv-files-into-one-dataframe-part-1/),[Part 2](https://datamariano.netlify.app/blog/2022-09-14-merging-spreadsheet-or-csv-files-into-one-dataframe-part-2/) and [Part 3](https://datamariano.netlify.app/blog/2022-09-19-merging-spreadsheet-or-csv-files-into-one-dataframe-part-3/) to address these issues. We will use the same workflow as before, adding a few more steps in order to have proper datetime data type and taking into account lags caused by switching to summer time. \
\
We start again with loading the packages we will use, continue with setting the working directory, get all filenames we want to read with the `list.files()` function and print our `filenames` to check them. For this part we use files with ".xlsx" extension as in Part 3.


```r
library(dplyr)
library(readxl)
library(tibble)
library(stringr)
library(lubridate)
library(tidyr)
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
##  [1] "JeanCojon_20150121.xlsx"      "JeanCojon_20150224.xlsx"     
##  [3] "JeanCojon_20150318.xlsx"      "JeanCojon_20150417.xlsx"     
##  [5] "JeanCojon_20150505.xlsx"      "JeanCojon_20150528.xlsx"     
##  [7] "JeanCojon_20150702.xlsx"      "JeanCojon_20150723.xlsx"     
##  [9] "JeanCojon_20150812.xlsx"      "JeanCojon_20150826.xlsx"     
## [11] "JeanCojon_20150917.xlsx"      "JeanCojon_20151009.xlsx"     
## [13] "JeanCojon_20151106.xlsx"      "JeanCojon_20151208.xlsx"     
## [15] "JeanCojon_20160107.xlsx"      "portobello-road20150122.xlsx"
## [17] "portobello-road20150310.xlsx" "portobello-road20150522.xlsx"
## [19] "portobello-road20150723.xlsx" "portobello-road20150915.xlsx"
## [21] "portobello-road20151027.xlsx" "portobello-road20160108.xlsx"
```
After getting our `logger_names`, this time we create new, `clean_logger_names` too with another `str_replace()` to turn the "-" to "_", avoiding problems later. We missed this step in Part 3, but it will make our life easier when checking if everything was OK after our loop, because R has some restrictions for object names. It can work when simply reading, transforming and exporting data with our loop, but if we wanted to call our object after an assignment, we would have to use its name surrounded by `` (backticks).


```r
logger_names <- unique(str_replace(filenames, "([^_]+)(?:_)*\\d{8}.+", "\\1")) 
clean_logger_names <- logger_names %>% str_replace("-", "_")
```


```r
clean_logger_names
```

```
## [1] "JeanCojon"       "portobello_road"
```
We create the usual `actual_year` and `real_time` objects to aid our workflow.




```r
actual_year <- as.integer(str_extract(getwd(), "\\d{4}"))
```


```r
real_time <- as_tibble_col(seq.POSIXt(make_datetime(actual_year, 1, 1, 0, 0, 0), 
                                  make_datetime(actual_year, 12, 31, 23, 55, 0), 600), 
                       column_name = "Datetime")
```

In the next step we read one sample file with summer time and one with winter time at autumn change to see the main problem related to their switch: there is an overlap between older and newer values, because an extra hour is added to summer time values. At the spring change there is only a gap that is easier to handle, but our solution will solve both switches.


```r
a_gmt2_sample_file <- read_xlsx("JeanCojon_20151106.xlsx") %>% 
      select(starts_with(c("Date", "Unit")))

a_gmt1_sample_file <- read_xlsx("JeanCojon_20151208.xlsx") %>% 
      select(starts_with(c("Date", "Unit")))
```

Let's inspect the name of their first column that contains the datetime values.


```r
colnames(a_gmt2_sample_file)[1]
```

```
## [1] "Date Time, GMT+02:00"
```

```r
colnames(a_gmt1_sample_file)[1]
```

```
## [1] "Date Time, GMT+01:00"
```

So checking the datetime column names we see that "GMT+02:00" for summer time and "GMT+01:00" for winter time are indicated there and we can use that to automate the process. But before moving on, we take a look at the last rows of the sample file with GMT+2 and the first rows of the sample file with GMT+1 time to see the overlapping datetime values.


```r
tail(a_gmt2_sample_file, 10)
```

```
## # A tibble: 10 x 5
##    `Date Time, GMT+02:00` `Unit, Y (serial n~` `Unit, Y (seri~` `Unit, Y (seri~`
##    <chr>                  <chr>                <chr>            <chr>           
##  1 11.06.15. 12:20:00     27.8881315886974     9.86417709849775 49.8240498173982
##  2 11.06.15. 12:30:00     21.5640082210302     23.1103064678609 -10.13070944696~
##  3 11.06.15. 12:40:00     88.8741133827716     11.203750288114  6.52594136074185
##  4 11.06.15. 12:50:00     22.1660946309566     87.5381153263152 7.71790613420308
##  5 11.06.15. 13:00:00     92.3874419927597     7.90100204758346 38.0318613816053
##  6 11.06.15. 13:10:00     73.1582580599934     89.9912063032389 45.7896863482893
##  7 11.06.15. 13:20:00     -3.09232463128865    10.4290564917028 98.2516137324274
##  8 11.06.15. 13:30:00     76.3282382395118     32.5180068705231 35.9746613260359
##  9 11.06.15. 13:40:00     54.1597123630345     56.1406901292503 0.9899814520031~
## 10 11.06.15. 13:50:00     43.2433468941599     97.7455416321754 97.0103600155562
## # ... with 1 more variable: `Unit, Y (serial no: 10987): Sensor_4` <chr>
```

```r
head(a_gmt1_sample_file, 10)
```

```
## # A tibble: 10 x 5
##    `Date Time, GMT+01:00` `Unit, Y (serial n~` `Unit, Y (seri~` `Unit, Y (seri~`
##    <chr>                  <chr>                <chr>            <chr>           
##  1 11.06.15. 13:00:00     19.0152423735708     -15.38976636715~ 14.6211184002459
##  2 11.06.15. 13:10:00     44.9437055271119     98.3004723489285 62.7803751360625
##  3 11.06.15. 13:20:00     11.1641562730074     17.4884293694049 7.54246301017702
##  4 11.06.15. 13:30:00     9.46030540391803     47.6624365337193 42.6431741844863
##  5 11.06.15. 13:40:00     -19.9923214223236    59.1184365842491 99.7097657062113
##  6 11.06.15. 13:50:00     89.4842337351292     30.0882528442889 -6.845863498747~
##  7 11.06.15. 14:00:00     83.055692082271      44.5414990372956 73.2836519181728
##  8 11.06.15. 14:10:00     23.4187546372414     10.6995169259608 93.6534661054611
##  9 11.06.15. 14:20:00     83.8505972735584     -11.850414853543 38.5226809605956
## 10 11.06.15. 14:30:00     90.0042162463069     -12.07521093077~ 98.5488815046847
## # ... with 1 more variable: `Unit, Y (serial no: 10987): Sensor_4` <chr>
```

As we can see there are the same datetime values at the end of the earlier file as at the beginning of the later. The different values of "Unit" columns between the two files for the same times prove that these rows are not duplicates. We may not see that clearly as a result of 2 digit year format, but our dates are in month-day-year format, because this download happened on 6th November in 2015. It means that data collection of the first file ended and the recording of the second file began on that day. We usually have to deal with various datetime formats, like in the case of our other series of files, so let's check one of these too.


```r
a_ymd_hms_sample_file <- read_xlsx("portobello-road20150522.xlsx") %>% 
  select(starts_with(c("Date", "Unit")))

head(a_ymd_hms_sample_file)
```

```
## # A tibble: 6 x 3
##   `Date Time, GMT+01:00` `Unit, X (serial no: 556677): Sensor~` `Unit, X (seri~`
##   <chr>                  <chr>                                  <chr>           
## 1 15.03.10. 12:20:00     -6.30916230380535                      27.5114460103214
## 2 15.03.10. 12:30:00     6.54009780846536                       0.6494624447077~
## 3 15.03.10. 12:40:00     21.9419079180807                       40.678947288543 
## 4 15.03.10. 12:50:00     96.8136007618159                       13.8362905196846
## 5 15.03.10. 13:00:00     17.7775215171278                       77.6639157999307
## 6 15.03.10. 13:10:00     10.633775787428                        92.1126830670983
```

Data series of this file started on 10th March in 2015, so there is year-month-day format in this file. Therefore our loop must handle this difference between ymd-mdy formats.\
\
Talking about the loop, let's review our modifications compared to earlier versions. \
As previously, we define the `files_for_one_logger`, followed by initiating an empty `list_for_dataframes`, then an inner loop over `files_for_one_logger` to read the files into the list as `data.frame`s and binding their rows into a `combined_df`, then `left_join()` it with `real_time` to get our `final_df` that we `assign()` into an individual `data.frame` object using the `logger_names` and `actual_year` vectors. \
\
Again, in the reading pipeline we select and rename our columns, but this time we use the more up-to-date `rename_with()` instead of `rename_at()` *(as it has remained here from earlier solutions along with some other functions, but we will discuss them in a later post)*. We mentioned above that files with different datetime formats must be distinguished, so the loop contains an additional `if` statement to detect files based on logger name then use either `ymd_hms()` or `mdy_hms()` to turn the character data type to datetime. Additionally, we apply the `round.POSIXt()` function to round the occasionally fragment values to whole seconds. These may not be visible easily, but sometimes present in our data and can lead to unsuccessful joins because of mismatching values. The rounding unit can be set to a couple of time units, but we use the default seconds to see if there is any other issue with our values. We also turn the other columns into numeric type.\
\
When binding the rows of our `list_for_dataframes` we start a new pipeline to solve the GMT+1-GMT+2 problem. By applying `pivot_longer()` we turn our datetime column into two distinct columns: one column ("gmt") containing the former column name in all rows and another ("dt_to_correct") with the actual datetime values. The latter is actually the same datetime series under a new name, but this way we have the GMT+1 or GMT+2 information in all rows too, stored in the former column name. After filtering out NAs as a security step, we can use these new columns when defining the +1 and +2 cases of GMT in a `case_when()` function, wrapped inside a `mutate()` call. We simply subtract one hour from values with GMT+2 and keep values with GMT+1 to create our new "Datetime" column. Then we select (and by that reorder) this Datetime and our Sensor columns, ahead of filtering for the `actual_year`. \
\
And to get our `final_df` we have to `left_join()` our `combined_df` with `real_time`, before `assign()` our `final_df`s into individual `data.frame`s. Finally we close the loop by removing temporary objects.


```r
for (name in seq_along(logger_names)){ 
  
  files_for_one_logger <- list.files(pattern = str_c("^", logger_names[name], ".*.xlsx"))

  list_for_dataframes <- list()

  if (str_detect(logger_names[name], "portobello")){
  for (i in seq_along(files_for_one_logger)) {
    list_for_dataframes[[i]] <- read_xlsx(files_for_one_logger[i]) %>% 
      select(starts_with(c("Date", "Unit"))) %>% 
      rename_with(~str_replace(., ".*(Sensor_.{1})", "\\1"), starts_with("Unit")) %>%
      mutate(across(contains("Date"), ymd_hms), across(contains("Date"), round.POSIXt),
             across(is.character, as.numeric))
    }
  } else {
    for (i in seq_along(files_for_one_logger)) {
    list_for_dataframes[[i]] <- read_xlsx(files_for_one_logger[i]) %>% 
      select(starts_with(c("Date", "Unit"))) %>% 
      rename_with(~str_replace(., ".*(Sensor_.{1})", "\\1"), starts_with("Unit")) %>%
      mutate(across(contains("Date"), mdy_hms), , across(contains("Date"), round.POSIXt),
             across(is.character, as.numeric))
    }
  }

  combined_df <- bind_rows(list_for_dataframes) %>% 
    pivot_longer(contains("Date"), names_to = "gmt", values_to = "dt_to_correct") %>% 
    filter(!is.na(dt_to_correct)) %>%
    mutate(Datetime = case_when(str_detect(gmt, "\\+01") ~ dt_to_correct, 
                                str_detect(gmt, "\\+02") ~ dt_to_correct-hours(1))) %>% 
    select(Datetime, starts_with("Sensor")) %>% 
    filter(year(Datetime)==actual_year)
  
  final_df <- left_join(real_time, combined_df, by="Datetime")
  
  assign(str_c(clean_logger_names[name], "_", actual_year), final_df)

 rm(combined_df, final_df, i, name, files_for_one_logger)
 
}
```

check former overlapping rows
that's why we needed clean names



```r
JeanCojon_2015 %>% filter(Datetime>=ymd_hm("2015-11-06 12:00"), Datetime<=ymd_hm("2015-11-06 14:00"))
```

we can now export or work further with these dfs
