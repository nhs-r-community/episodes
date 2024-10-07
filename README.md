
<!-- README.md is generated from README.Rmd. Please edit that file -->

# NHSRepisodes

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://lifecycle.r-lib.org/articles/stages.html#experimental)
[![R-CMD-check](https://github.com/nhs-r-community/NHSRepisodes/actions/workflows/R-CMD-check.yaml/badge.svg)](https://github.com/nhs-r-community/NHSRepisodes/actions/workflows/R-CMD-check.yaml)
<!-- badges: end -->

## Installation instructions

You can install the development version of this package from GitHub
with:

``` r
# install.packages("remotes")
remotes::install_github("https://github.com/nhs-r-community/NHSRepisodes")
```

To find out more about the functions there is a vignetted for [getting
started](https://nhs-r-community.github.io/NHSRepisodes/articles/NHSRepisodes.html).

## Motivation

***NHSRepisodes*** is a (hopefully) temporary solution to a small
inconvenience that relates to
[data.table](https://cran.r-project.org/package=data.table),
[dplyr](https://cran.r-project.org/package=dplyr) and
[ivs](https://cran.r-project.org/package=ivs); namely that dplyr is
currently [slow when working with a large number of
groupings](https://github.com/tidyverse/dplyr/issues/5017) and
data.table [does not easily support the record
class](https://github.com/Rdatatable/data.table/issues/4910) on which
ivs intervals are based.

To expand on issues consider the following small set of episode data:

``` r
library(NHSRepisodes)
library(dplyr)
library(ivs)
library(data.table)

# note - we need functionality introduced in dplyr 1.1.0.
if (getNamespaceVersion("dplyr") < "1.1.0") {
    warning("Please update dplyr to version 1.1.0 or higher to run these examples.")
    knitr::knit_exit()
}

# Let's note the package versions used in generating this README
packages <- c("NHSRepisodes", "dplyr", "data.table", "ivs")
mutate(tibble(packages), version = sapply(packages, getNamespaceVersion))
#> # A tibble: 4 × 2
#>   packages     version   
#>   <chr>        <chr>     
#> 1 NHSRepisodes 0.1.0.9000
#> 2 dplyr        1.1.4.9000
#> 3 data.table   1.16.0    
#> 4 ivs          0.2.0

# Create a dummy data set give the first and last dates of an episode
dat <- tribble(
    ~id, ~start, ~end,
    1L, "2020-01-01", "2020-01-10",
    1L, "2020-01-03", "2020-01-10",
    2L, "2020-04-01", "2020-04-30",
    2L, "2020-04-15", "2020-04-16",
    2L, "2020-04-17", "2020-04-19",
    1L, "2020-05-01", "2020-10-01",
    1L, "2020-01-01", "2020-01-10",
    1L, "2020-01-11", "2020-01-12",
)

# This will create an object called dat and also open in the console
(dat <- mutate(dat, across(start:end, as.Date)))
#> # A tibble: 8 × 3
#>      id start      end       
#>   <int> <date>     <date>    
#> 1     1 2020-01-01 2020-01-10
#> 2     1 2020-01-03 2020-01-10
#> 3     2 2020-04-01 2020-04-30
#> 4     2 2020-04-15 2020-04-16
#> 5     2 2020-04-17 2020-04-19
#> 6     1 2020-05-01 2020-10-01
#> 7     1 2020-01-01 2020-01-10
#> 8     1 2020-01-11 2020-01-12
```

The {ivs} package provides an elegant way to find the minimum spanning
interval across these episodes:

``` r
dat |>
    mutate(interval = iv(start = start, end = end + 1)) |>
    reframe(interval = iv_groups(interval, abutting = FALSE), .by = id)
#> # A tibble: 4 × 2
#>      id                 interval
#>   <int>               <iv<date>>
#> 1     1 [2020-01-01, 2020-01-11)
#> 2     1 [2020-01-11, 2020-01-13)
#> 3     1 [2020-05-01, 2020-10-02)
#> 4     2 [2020-04-01, 2020-05-01)
```

Note that {ivs} creates intervals that are *right-open* meaning they are
inclusive on the left (have an opening square bracket `[`) and exclusive
on the right (with a closing a rounded bracket `)`). Consequently, in
our first call to `mutate()` we added 1 to the `end` value. This ensures
that the full range of dates are considered (e.g. for the first row we
want to consider all days from `2020-01-01` to `2020-01-10` not only up
until `2020-01-09`).

This works great when we only have a small number of ids to group by.
However, it becomes noticeably slow for a larger number:

``` r
# Creating a larger data set
n <- 125000
id2 <- sample(seq_len(n), size = n * 5, replace = TRUE)
start2 <- as.Date("2020-01-01") + sample.int(365, size = n * 5, replace = TRUE)
end2 <- start2 + sample(1:100, size = n * 5, replace = TRUE)

# creates the object big_dat and shows the first 10 rows as a tibble in the console
(big_dat <- tibble(id = id2, start = start2, end = end2))
#> # A tibble: 625,000 × 3
#>       id start      end       
#>    <int> <date>     <date>    
#>  1 90983 2020-02-07 2020-03-18
#>  2 59640 2020-07-05 2020-08-13
#>  3 29543 2020-04-05 2020-05-28
#>  4 17962 2020-06-07 2020-07-21
#>  5 13032 2020-08-25 2020-09-08
#>  6 94348 2020-05-11 2020-06-07
#>  7  7770 2020-03-18 2020-04-03
#>  8  9570 2020-09-25 2020-12-04
#>  9 86687 2020-11-04 2021-01-09
#> 10 66213 2020-04-09 2020-04-16
#> # ℹ 624,990 more rows

# checking the time to run
system.time(
    out_dplyr <- 
        big_dat |>
        mutate(interval = iv(start, end + 1)) |>
        reframe(interval = iv_groups(interval, abutting = FALSE), .by = id)
)
#>    user  system elapsed 
#>  13.870   0.065  13.996
```

If you were not already using it, this is likely the time you would
reach for the {data.table} package. Unfortunately the interval class
created by {ivs} is built upon on the [record type from
vctrs](https://vctrs.r-lib.org/reference/new_rcrd.html), and this class
is not supported in {data.table}:

``` r
DT <- as.data.table(big_dat)
DT[, interval := iv(start, end + 1)]
#> Error in `[.data.table`(DT, , `:=`(interval, iv(start, end + 1))): Supplied 2 items to be assigned to 625000 items of column 'interval'. If you wish to 'recycle' the RHS please use rep() to make this intent clear to readers of your code.
```

We can go through a few more steps to get a comparable answer but still
find slightly slower performance:

``` r
fun <- function(s, e) {
    interval <- iv(s, e)
    groups <- iv_groups(interval, abutting = FALSE)
    list(start = iv_start(groups), end = iv_end(groups))
}

system.time(out_dt <- DT[, fun(start, end + 1), by = id])
#>    user  system elapsed 
#>  14.972   0.022  14.984
```

***NHSRepisodes*** solves this with the `merge_episodes()` function:

``` r
merge_episodes(big_dat)
#> # A tibble: 334,767 × 4
#>       id .interval_number .episode_start .episode_end
#>    <int>            <int> <date>         <date>      
#>  1     1                1 2020-04-16     2020-06-23  
#>  2     2                1 2020-01-12     2020-02-22  
#>  3     2                2 2020-07-22     2020-08-16  
#>  4     2                3 2020-09-27     2021-01-11  
#>  5     3                1 2020-01-06     2020-03-02  
#>  6     3                2 2020-03-08     2020-03-16  
#>  7     3                3 2020-06-05     2020-08-26  
#>  8     4                1 2020-02-05     2020-05-03  
#>  9     4                2 2020-05-19     2020-09-08  
#> 10     5                1 2020-01-14     2020-03-13  
#> # ℹ 334,757 more rows

# And for comparison with earlier timings
system.time(out <- merge_episodes(big_dat))
#>    user  system elapsed 
#>   0.917   0.000   0.335

# equal output (subject to ordering)
out <- out |> 
    mutate(interval = iv(start = .episode_start, end = .episode_end + 1)) |> 
    select(id, interval)

out_dplyr <- arrange(out_dplyr, id, interval)

out_dt <- out_dt |> 
    as.data.frame() |> 
    as_tibble() |> 
    mutate(interval = iv(start = start, end = end)) |> 
    select(id, interval) |> 
    arrange(id, interval)

all.equal(out, out_dplyr)
#> [1] TRUE
all.equal(out, out_dt)
#> [1] TRUE
```

We also provide another function `add_parent_interval()` that associates
the the minimum spanning interval with each observation without reducing
to the unique values:

``` r
add_parent_interval(dat)
#> # A tibble: 8 × 6
#>      id start      end        .parent_start .parent_end .interval_number
#>   <int> <date>     <date>     <date>        <date>                 <int>
#> 1     1 2020-01-01 2020-01-10 2020-01-01    2020-01-10                 1
#> 2     1 2020-01-03 2020-01-10 2020-01-01    2020-01-10                 1
#> 3     2 2020-04-01 2020-04-30 2020-04-01    2020-04-30                 1
#> 4     2 2020-04-15 2020-04-16 2020-04-01    2020-04-30                 1
#> 5     2 2020-04-17 2020-04-19 2020-04-01    2020-04-30                 1
#> 6     1 2020-05-01 2020-10-01 2020-05-01    2020-10-01                 3
#> 7     1 2020-01-01 2020-01-10 2020-01-01    2020-01-10                 1
#> 8     1 2020-01-11 2020-01-12 2020-01-11    2020-01-12                 2
```
