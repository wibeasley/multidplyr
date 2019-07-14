
<!-- README.md is generated from README.Rmd. Please edit that file -->

# multidplyr

<!-- badges: start -->

[![Travis build
status](https://travis-ci.org/tidyverse/multidplyr.svg?branch=master)](https://travis-ci.org/tidyverse/multidplyr)
[![Codecov test
coverage](https://codecov.io/gh/tidyverse/multidplyr/branch/master/graph/badge.svg)](https://codecov.io/gh/tidyverse/multidplyr?branch=master)
[![CRAN
status](https://www.r-pkg.org/badges/version/multidplyr)](https://cran.r-project.org/package=multidplyr)
[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
<!-- badges: end -->

## Overview

multidplyr is a backend for dplyr that partitions a data frame across
multiple cores. You tell multidplyr how to split the data up with
`partition()` and then the data stays on each node until you explicitly
retrieve it with `collect()`. This minimises the amount of time spent
moving data around, and maximises parallel performance. This idea is
inspired by [partools](http://bit.ly/1Nve8v5) by Norm Matloff and
[distributedR](http://bit.ly/1KZVAwK) by the Vertica Analytics team.

Due to the overhead associated with communicating between the nodes, you
won’t expect to see much performance improvement on basic dplyr verbs
with less than \~10 million observations. However, you’ll see
improvements much faster if you’re doing more complex operations with
`do()`.

(Note that unlike other packages in the tidyverse, multidplyr requires R
3.5 or greater. We hope to relax this requirement [in the
future](https://github.com/traversc/qs/issues/11).)

## Installation

To install from GitHub:

``` r
# install.packages("devtools")
devtools::install_github("tidyverse/multidplyr")
```

## Usage

To use multidplyr, you first create a cluster of the desired number of
workers. Each one of these workers is a separate R process, and the
operating system will spread their execution across multiple cores:

``` r
library(multidplyr)
library(dplyr, warn.conflicts = FALSE)

cluster <- new_cluster(4)
```

There are two primary ways to use multidplyr. The first, and most,
efficient is to load a different data on each worker:

``` r
# Create a filename vector containing different values on each worker
cluster_assign_each(cluster, "filename",
  list("a.csv", "b.csv", "c.csv", "d.csv")
)

# Use vroom to quickly load the csvs
cluster_assign(cluster, "my_data", vroom::vroom(filename))

# Create a party_df usingt the my_data variable on each worker
my_data <- party_df(cluster, "my_data")
```

Alternatively, if you already have the data loaded in the main session,
you can use `partition()` to automatically spread it across groups.
Specify one or more partitioning variables to ensure all of the
observations belonging to that group end up on the same worker. This
will happen automatically on grouped data:

``` r
library(nycflights13)

flight_dest <- flights %>% group_by(dest) %>% partition(dest, .cluster = cluster)
#> Warning: group_indices_.grouped_df ignores extra arguments
flight_dest
#> Source: party_df [336,776 x 19]
#> Groups: dest
#> Shards: 4 [81,594--86,548 rows]
#> 
#> # Description: party_df
#>    year month   day dep_time sched_dep_time dep_delay arr_time
#>   <int> <int> <int>    <int>          <int>     <dbl>    <int>
#> 1  2013     1     1      544            545        -1     1004
#> 2  2013     1     1      558            600        -2      923
#> 3  2013     1     1      559            600        -1      854
#> 4  2013     1     1      602            610        -8      812
#> 5  2013     1     1      602            605        -3      821
#> 6  2013     1     1      611            600        11      945
#> # … with 3.368e+05 more rows, and 12 more variables: sched_arr_time <int>,
#> #   arr_delay <dbl>, carrier <chr>, flight <int>, tailnum <chr>,
#> #   origin <chr>, dest <chr>, air_time <dbl>, distance <dbl>, hour <dbl>,
#> #   minute <dbl>, time_hour <dttm>
```

Now you can work with it like a regular data frame, but the computations
will be spread across multiple cores. Once you’ve finished computation,
use `collect()` to bring the data back to the host session:

``` r
flight_dest %>% 
  summarise(delay = mean(dep_delay, na.rm = TRUE), n = n()) %>% 
  collect()
#> # A tibble: 105 x 3
#>    dest  delay     n
#>    <chr> <dbl> <int>
#>  1 ABQ    13.7   254
#>  2 AUS    13.0  2439
#>  3 BQN    12.4   896
#>  4 BTV    13.6  2589
#>  5 BUF    13.4  4681
#>  6 CLE    13.4  4573
#>  7 CMH    12.2  3524
#>  8 DEN    15.2  7266
#>  9 DSM    26.2   569
#> 10 DTW    11.8  9384
#> # … with 95 more rows
```

Note that there is some overhead associated with copying data from the
worker nodes back to the host node (and vice versa), so you’re best off
using multidplyr with more complex operations. See
`vignette("multidplyr")` for more details.
