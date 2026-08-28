# NYC Flights: Relational Data & Delay Analysis

**Author:** Agne Pack
**Tools:** R, dplyr, ggplot2, ggmap, nycflights13
**Type:** Independent statistical analysis (coursework, cleaned up for portfolio use)

## Overview

This project works with the `nycflights13` dataset — five related tables covering
every flight departing New York City airports in 2013, along with airline, airport,
plane, and hourly weather data. The focus is on relational-data techniques: joining
across multiple tables to answer questions no single table could answer on its own,
then visualizing the results geographically.

## Purpose

1. Practice and demonstrate relational-data operations (`left_join`, `inner_join`,
   `semi_join`, `anti_join`) across a genuinely multi-table dataset, rather than a
   single flat file.
2. Map flight destinations and visualize arrival delay by airport during a known
   severe-weather event.
3. Begin exploring whether plane age relates to flight delay, and identify data-
   quality issues (missing tail numbers, unmatched airport codes) that would need
   to be resolved before that question could be answered rigorously.

## Data

`nycflights13` (via the R package of the same name) provides five linked tables:

| Table | Rows | Key fields |
|---|---|---|
| `flights` | 336,776 | year/month/day, dep/arr times & delays, carrier, tailnum, origin, dest |
| `airlines` | 16 | carrier code → airline name |
| `airports` | 1,458 | faa code, name, lat/lon, altitude, timezone |
| `planes` | 3,322 | tailnum, manufacture year, model, engines, seats |
| `weather` | 26,115 | hourly conditions (temp, wind, precip, visibility) by origin airport |

## Method

```r
library(tidyverse)
library(nycflights13)
library(ggmap)
library(maps)
library(mapdata)

# Map every airport that appears as a flight destination
airports %>%
  semi_join(flights, c("faa" = "dest")) %>%
  ggplot(aes(lon, lat)) +
  borders("state") +
  geom_point() +
  coord_quickmap()

# Attach both origin and destination coordinates to every flight
flights %>%
  left_join(airports, by = c("dest" = "faa")) %>%
  left_join(airports, by = c("origin" = "faa"), suffix = c(".dest", ".origin")) %>%
  select(dest, origin, contains("lat"), contains("lon"))

# Average arrival delay by destination airport on a known severe-weather day
flights %>%
  filter(year == 2013, month == 6, day == 13) %>%
  group_by(dest) %>%
  summarise(delay = mean(arr_delay, na.rm = TRUE)) %>%
  inner_join(airports, by = c("dest" = "faa")) %>%
  ggplot(aes(y = lat, x = lon, size = delay, colour = delay)) +
  borders("state") +
  geom_point() +
  coord_quickmap()

# Exploring plane age vs. delay: average delay by tail number
flights %>%
  group_by(tailnum) %>%
  summarise(
    avg_dep_delay = mean(dep_delay, na.rm = TRUE),
    avg_arr_delay = mean(arr_delay, na.rm = TRUE)
  )

# Data-quality checks
flights %>% filter(is.na(tailnum))                      # flights with no tail number on record
flights %>% filter(!is.na(dep_delay)) %>%
  group_by(tailnum) %>% summarize(n = n()) %>% filter(n > 100)  # planes with enough flights to trend
anti_join(flights, airports, by = c("dest" = "faa"))     # destinations missing from the airports table
anti_join(airports, flights, by = c("faa" = "dest"))     # airports never used as a destination
```

*(Chart images are included alongside this README in the project folder.)*

## Key Findings

- **Semi-join mapping:** Plotting only airports that actually appear as flight
  destinations (via `semi_join`) correctly filtered the 1,458-airport reference
  table down to the much smaller set NYC flights actually reach — including two
  clear outliers far outside the continental US (Alaska and Hawaii).
- **Weather-delay event:** Isolating June 13, 2013 — a day with recorded severe
  thunderstorms and two tornadoes — and mapping average arrival delay by
  destination showed a visible cluster of longer delays concentrated in the
  Southeast/mid-Atlantic, consistent with the reported weather disruption.
- **Missing-data checks:** 2,512 flights had no recorded tail number, meaning
  plane-level analysis (like the age-vs-delay question) can only ever cover a
  subset of flights. This matters — any conclusion about plane age would need to
  account for that gap rather than silently drop it.
- **Unmatched airports:** `anti_join(flights, airports, by = c("dest" = "faa"))`
  found 7,602 flights whose destination code doesn't exist in the airports
  reference table at all (largely small or military airfields not included in
  the reference data) — a reminder that "the join worked" and "the join is
  complete" are two different things worth checking separately.

## Conclusions

The relational structure of this dataset is the whole point: no single table
answers "which airports were hit hardest by the June 13 storms" or "how many
flights can't be tied to a specific aircraft" — those answers only emerge once
`flights` is joined against `airports` and `planes`. The weather-delay map is the
clearest result here, visibly tying a known real-world event to a spike in average
delay concentrated in the right region.

The plane-age-vs-delay question was set up but not carried to a full conclusion in
this version of the project — the `anti_join` and missing-tailnum checks above were
the necessary first step (confirming how much of the dataset can even be used for
that comparison) before fitting any real model. That's a natural next step if this
project gets revisited: join `flights` to `planes` on `tailnum`, exclude the
~2,500 flights with no tail number, and test delay against `year` (as a proxy for
plane age).

## References

Wickham, H., & Grolemund, G. *R for Data Science*, Chapter 13: Relational Data.
https://r4ds.had.co.nz/relational-data.html

Li, A. *R Spatial Workshop Notes*, Chapter 5: R Markdown and Custom Maps.
https://spatialanalysis.github.io/workshop-notes/r-markdown-and-custom-maps.html

Arnold, J. *r4ds Exercise Solutions*: Relational Data.
https://github.com/jrnold/r4ds-exercise-solutions/blob/master/relational-data.Rmd

`nycflights13` dataset, accessed via RStudio Cloud, March 2021.
