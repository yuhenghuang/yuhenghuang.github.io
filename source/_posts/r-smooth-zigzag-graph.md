---
title: R - Smooth Zigzag Graph
top: false
cover: false
toc: true
mathjax: false
date: 2020-05-22 22:19:14
password:
summary:
tags:
  - Tidyverse
  - Interpolation
categories: R
---

Visualization shall be the most important part in the field of data science. Despite there are many positions for hard-core machine learning techniques, in the future it is expected to see more and more jobs for so-called general purpose data science, where data scientists are facing clients who barely know mathematics and models rather than engineering, which could be solved by logics. The most straight forward approach of showing the results from a project to the client is always all sorts of visualizations (graphs) , with mathematical formulae playing the sub role. To tell the truth, clients could not care less about technical details as long as the theory (model) is autonomous in their minds. A well-drawn graph is exactly the most powerful weapon to give the autonomy to the others and convince them from inside of their hearts. 

There are many ways of making the visualizations more attractive and appealing, so many that even one book can not depict them all. This article is focusing on one specific techniques that will make your graph more appealing without deviating much from the data behind the graph.


### Prepare Data

A very good example of zigzag data is [Google Trends](https://trends.google.com/), where the search trend of a certain word/topic is visualized given various settings (e.g. location). Especially when the time range is set to `past 4 hours`, the unit of data will be `one minute`. The fluctuation of data can be quite large that a normal trend(line) plot looks very bleak.

For example, the data of `softbank` in `Japan`, `past 4 hours` are available in the following [link](https://trends.google.com/trends/explore?date=now%204-H&geo=JP&q=softbank).

Let's visualize the raw data in R.

```r
library(tidyverse)
library(haven)
library(lubridate)
library(imputeTS) # linear interpolation
library(smoother) # gaussian filter

theme_set(theme_bw())

# timestamp with timezone will be converted to utc time
df_origin <- read_csv('./multiTimeline.csv', col_names = c('date', 'hit'), skip = 3)

# limit time range to 2 hours (121 obs.)
start_at <- ceiling_date(min(df_origin$date), unit = 'hours') + hours(1)
end_at <- floor_date(max(df_origin$date), unit = 'hours')

df_main <- df_origin %>%
  filter(date>=start_at, date<=end_at) %>%
  mutate(date = date + hours(9)) # optional: convert to local(Japan) timezone

ggplot(df_main, aes(date, hit)) +
  geom_line() +
  theme(
    axis.title = element_text(size = 20),
    axis.text = element_text(size = 16)
  )
```

Without loss of generality, the columns are renamed to *date* and *hit*. To make the fluctuations more conspicuous, the time range is limited to a 2-hour interval.

![Raw Data](/images/r_smooth_zigzag_raw.jpg)

The data itself shall reflect the real trend while the zigzags make it look less realist in naked eyes. It would be positively better to smooth the line and maintain the trend of the data simultaneously.

### Smooth

In this section, we smooth the line by following steps
1. construct smaller unit (1 minute -> 6 seconds)
2. left join the original data
3. linearly interpolate missing values (NAs)
4. apply gaussian filter (adjustable window size)

```r
# execute after previous block
df_smooth <- tibble(
  date = seq.POSIXt(min(df_main$date), max(df_main$date), '6 sec') # convert unit
) %>%
  left_join(df_main, by='date') %>% # left join
  mutate(hit = na_interpolation(hit), # linear interpolation
         hit = smth.gaussian(hit, window = 21)) # gaussian filter

ggplot(df_smooth, aes(date, hit)) +
  geom_line() +
  theme(
    axis.title = element_text(size = 20),
    axis.text = element_text(size = 16)
  )
```


![Smoothed Data](/images/r_smooth_zigzag_smth.jpg)
<sup>* NAs generated at the boundaries after applying gaussian filter. The number is the same as window size.<sup>

With larger window size the line will be smoother, whereas it deviates more from original data. The trade-off between smoothness and deviations is up to you.