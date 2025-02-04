Cleaning and Pre-Processing Divvy Bikeshare Data
================
Andrew Luyt
<br>Last updated: Friday August 13, 2021

-   [Purpose](#purpose)
-   [Data Issues Resolved](#data-issues-resolved)
-   [New variables added](#new-variables-added)
-   [The cleaning code](#the-cleaning-code)

## Purpose

Transform the numerous raw .csv files into a single, cleaned .csv with
some extra features.

Read **all** csv files available (except `alldata.csv.gz`) into a single
dataframe, add features, then save complete dataset to `alldata.csv.gz`.

Only include files in the data directory which you wish to include in
the analysis. If datasets which are too old to be relevant are included
in the data directory they will also be read, undoubtedly to your great
despair.

``` r
library(data.table)  # use fread for speed
library(R.utils)     # allow fread to read/write .gz
library(tidyverse)
library(lubridate)
library(geosphere)   # calculate trip distances

# the "mode" from statistics - the most frequently occurring value
Mode <- function(x) {
  unique_vals <- unique(x)
  unique_vals[which.max(tabulate(match(x, unique_vals)))]
}
```

``` r
# Don't include trip_id (it's just a unique string ID) and dropping it
# cuts memory significantly.
# Use factors to reduce memory consumption on my poor laptop
df <-
  list.files(path = "./data",
             pattern = "*-divvy-tripdata.csv.gz",
             full.names = TRUE) %>%
  map_df(~fread(.,
                select= (2:13),
                colClasses = c(start_station_id = 'character',
                               end_station_id = 'character'),
                stringsAsFactors = TRUE))
```

## Data Issues Resolved

We drop rows to remove incomplete, probably-corrupt, or irrelevant data:

-   **Blank station information:** we need to know where trips start and
    end.
-   **Huge or tiny trip times:** are these bikes stolen, lost, or
    broken, then brought back into service? Did someone undock a bike,
    then change their mind? Is it some sort of internal system test?
    Whatever the cause, these outliers are irrelevant for casual vs
    member analysis.
    -   We filter out any trips longer than 24 hours or shorter than 1
        minute
-   **Impossible speeds:** remove any trips with an estimated speed over
    70kph. This represents fewer than 20 trips at time of writing.
-   Station names can have **multiple station IDs or names** (did they
    get re-numbered?) We collapse these into single IDs/names.
-   **Some station names are suffixed** with " (\*)" or other patterns.
    This suffix is removed.

## New variables added

-   `weekday`: by the start of the ride (1-7, starts Monday)
-   `weekend_weekday`: Convenience variable to help visualize weekend
    patterns
-   `month`: Abbreviated, by the start of the ride
-   Four `sector` variables: a rectangular area on the map. Created by
    rounding longitude & latitude to two decimal points.
    -   Interpreted as a geographical **area** or “bin” where a station
        exists, as opposed to a point. Meant as a convenient aggregating
        measure for later analysis.
-   `trip_minutes`: Some trips are of negative duration and will be
    filtered out later. Online research suggests that some of this comes
    from Divvy taking bikes in and out of service for quality control
    reasons.
-   `trip_delta_`: horizontal/vertical components of a trip (km). East
    and north are positive values, west and south are negative.
-   `is_round_trip`: a boolean flag that shows if the trip started and
    ended at the same station. Meant to distinguish between commute-type
    trips and pleasure cruises, and filter out trip distances of 0.
-   `trip_distance`: straight-line distance (km) from start to finish.
    This is a proxy for the actual trip path as ridden, which is not
    available in the dataset. It will always be smaller than reality.
    Round trips have a distance of 0 by this measure. Unavoidable, as
    GPS tracks showing the true route taken are not available.
-   `trip_kph`: Estimate of overall trip speed. Since `trip_distance` is
    an underestimate, this speed is faster than reality. However, it
    will be useful as a *relative comparison* between groups.

## The cleaning code

``` r
df <- df %>%
  mutate(
    trip_minutes = as.numeric(difftime(ended_at, started_at, units = "mins")),
    weekday = factor(lubridate::wday(started_at, week_start = 1),
                     levels = 1:7,
                     labels = c("Mon", "Tue", "Wed",
                                "Thu", "Fri", "Sat", "Sun")),
    weekend_weekday = if_else(weekday %in% c("Sat", "Sun"), "weekend", "weekday"),
    month = factor(lubridate::month(started_at, abbr = TRUE, label = TRUE)),
    is_round_trip = if_else(start_station_id == end_station_id,
                            TRUE, FALSE),
    # remove suffixes like (*) or (TEMP)
    start_station_name = str_replace(start_station_name, " \\(\\*\\)", ""),
    end_station_name = str_replace(end_station_name, " \\(\\*\\)", ""),
    start_station_name = str_replace(start_station_name, " \\([^()]{0,}\\)", ""),
    end_station_name = str_replace(end_station_name, "\\([^()]{0,}\\)", ""),
    # many stations IDs have duplicate names. Collapse them into one canonical name.
    start_station_name = str_replace(start_station_name, "McClurg Ct & Illinois St", "New St & Illinois St"),
    end_station_name = str_replace(end_station_name, "McClurg Ct & Illinois St", "New St & Illinois St"),
    start_station_name = str_replace(start_station_name, "Drake Ave & Fullerton Ave", "St. Louis Ave & Fullerton Ave"),
    end_station_name = str_replace(end_station_name, "Drake Ave & Fullerton Ave", "St. Louis Ave & Fullerton Ave"),
    start_station_name = str_replace(start_station_name, "Chicago Ave & Dempster St", "Dodge Ave & Main St"),
    end_station_name = str_replace(end_station_name, "Chicago Ave & Dempster St", "Dodge Ave & Main St"),
    start_station_name = str_replace(start_station_name, "Malcolm X College Vaccination Site", "Malcolm X College"),
    end_station_name = str_replace(end_station_name, "Malcolm X College Vaccination Site", "Malcolm X College"),
    start_station_name = str_replace(start_station_name, "Ashland Ave & 73rd St", "Ashland Ave & 74th St"),
    end_station_name = str_replace(end_station_name, "Ashland Ave & 73rd St", "Ashland Ave & 74th St"),
    start_station_name = str_replace(start_station_name, "Halsted St & 104th St", "Western Ave & 104th St"),
    end_station_name = str_replace(end_station_name, "Halsted St & 104th St", "Western Ave & 104th St"),
    start_station_name = str_replace(start_station_name, "Broadway & Wilson - Truman College Vaccination Site", "Broadway & Wilson Ave"),
    end_station_name = str_replace(end_station_name, "Broadway & Wilson - Truman College Vaccination Site", "Broadway & Wilson Ave"),
    start_station_name = str_replace(start_station_name, "HUBBARD ST BIKE CHECKING", "Base - 2132 W Hubbard Warehouse"),
    end_station_name = str_replace(end_station_name, "HUBBARD ST BIKE CHECKING", "Base - 2132 W Hubbard Warehouse"),
    start_station_name = str_replace(start_station_name, "Chicago Ave & Dempster St", "Dodge Ave & Main St"),
    end_station_name = str_replace(end_station_name, "Chicago Ave & Dempster St", "Dodge Ave & Main St")
    ) %>%
  filter(
    start_station_name != "",
    start_station_id != "",
    end_station_name != "",
    end_station_id != "",
    !is.na(start_lng),
    !is.na(start_lat),
    !is.na(end_lng),
    !is.na(end_lat),
    trip_minutes < 1440,
    trip_minutes > 1,
    # stations whose coordinates vary wildly
    start_station_id != 704,
    start_station_id != 709,
    start_station_id != "KA1503000055",
    start_station_id != 20123,
    end_station_id != 704,
    end_station_id != 709,
    end_station_id != "KA1503000055",
    end_station_id != 20123) %>%
  # GPS coords have some random error. Create canonical coordinates for each station.
  group_by(start_station_id) %>%
  mutate(start_lng = mean(start_lng),
         start_lat = mean(start_lat),
         start_station_name = first(start_station_name)) %>%
  group_by(end_station_id) %>%
  mutate(end_lng = mean(end_lng),
         end_lat = mean(end_lat),
         end_station_name = first(end_station_name)) %>%
  mutate(start_lng_sector = round(start_lng, digits = 2),
         start_lat_sector = round(start_lat, digits = 2),
         end_lng_sector = round(end_lng, digits = 2),
         end_lat_sector = round(end_lat, digits = 2)) %>%
  # distGeo prefers to work with matrices of the form lng1 lat1, lng2 lat2,
  # and we have those four columns, which we can cbind together.
  mutate(
    trip_distance = geosphere::distGeo(cbind(start_lng, start_lat), cbind(end_lng, end_lat)) / 1000,
    trip_delta_x = geosphere::distGeo(cbind(start_lng, start_lat), cbind(end_lng, start_lat)) / 1000,
    trip_delta_y = geosphere::distGeo(cbind(start_lng, start_lat), cbind(start_lng, end_lat)) / 1000,
    trip_delta_x =  # east positive, west negative
      if_else(end_lng > start_lng, trip_delta_x, (-1) * trip_delta_x),
    trip_delta_y =  # north positive, south negative
      if_else(end_lat > start_lat, trip_delta_y, (-1) * trip_delta_y),
    trip_kph = trip_distance / trip_minutes * 60) %>%
  filter(trip_kph < 70) %>%
  # some stations were renamed - choose one canonical name
  group_by(start_station_name) %>%
  mutate(start_station_id = Mode(start_station_id)) %>%
  group_by(end_station_name) %>%
  mutate(end_station_id = Mode(end_station_id)) %>%
  ungroup()

fwrite(x = df, file = "./data/alldata.csv.gz")
```
