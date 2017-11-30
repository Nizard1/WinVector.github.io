DplyrDependencies
================
Win-Vector LLC
11/30/2017

In [an earlier note](https://github.com/WinVector/Examples/blob/master/dplyr/Dependencies.md) we exhibited a non-signalling result corruption in `dplyr` `0.7.4`. In this note we demonstrate the [`seplyr`](https://winvector.github.io/seplyr/) work-around.

Re-establish up our example:

``` r
packageVersion("dplyr")
```

    ## [1] '0.7.4'

``` r
my_db <- DBI::dbConnect(RSQLite::SQLite(),
                        ":memory:")
d <- dplyr::copy_to(my_db, 
                    data.frame(valuesA = c("A", NA, NA),
                               valuesB = c("B", NA, NA),
                               canUseFix1 = c(TRUE, TRUE, FALSE),
                               fix1 = c('Fix_1_V1', "Fix_1_V2", "Fix_1_V3"),
                               canUseFix2 = c(FALSE, FALSE, TRUE),
                               fix2 = c('Fix_2_V1', "Fix_2_V2", "Fix_2_V3"),
                               stringsAsFactors = FALSE),
                    'd', 
                    temporary = TRUE, overwrite = TRUE)
knitr::kable(dplyr::collect(d))
```

| valuesA | valuesB |  canUseFix1| fix1       |  canUseFix2| fix2       |
|:--------|:--------|-----------:|:-----------|-----------:|:-----------|
| A       | B       |           1| Fix\_1\_V1 |           0| Fix\_2\_V1 |
| NA      | NA      |           1| Fix\_1\_V2 |           0| Fix\_2\_V2 |
| NA      | NA      |           0| Fix\_1\_V3 |           1| Fix\_2\_V3 |

[`seplyr`](https://winvector.github.io/seplyr/) has a fix/work-around for the earlier issue: automatically break up the steps into safe blocks ([announcement](http://www.win-vector.com/blog/2017/11/win-vector-llc-announces-new-big-data-in-r-tools/); here we are using the development [`seplyr`](https://winvector.github.io/seplyr/) `0.5.1` version of [`mutate_se()`](https://winvector.github.io/seplyr/reference/mutate_se.html)).

``` r
library("seplyr")
```

    ## Loading required package: wrapr

``` r
packageVersion("seplyr")
```

    ## [1] '0.5.1'

``` r
d %.>% 
  mutate_nse(., 
             valuesA := ifelse(is.na(valuesA) & canUseFix1, 
                               fix1, valuesA),
             valuesA := ifelse(is.na(valuesA) & canUseFix2, 
                               fix2, valuesA),
             valuesB := ifelse(is.na(valuesB) & canUseFix1, 
                               fix1, valuesB),
             valuesB := ifelse(is.na(valuesB) & canUseFix2, 
                               fix2, valuesB),
             mutate_nse_printPlan = TRUE) %.>% 
  select_se(., c("valuesA", "valuesB")) %.>% 
  dplyr::collect(.) %.>% 
  knitr::kable(.)
```

    ## $group00001
    ##                                              valuesA 
    ## "ifelse(is.na(valuesA) & canUseFix1, fix1, valuesA)" 
    ##                                              valuesB 
    ## "ifelse(is.na(valuesB) & canUseFix1, fix1, valuesB)" 
    ## 
    ## $group00002
    ##                                              valuesA 
    ## "ifelse(is.na(valuesA) & canUseFix2, fix2, valuesA)" 
    ##                                              valuesB 
    ## "ifelse(is.na(valuesB) & canUseFix2, fix2, valuesB)"

| valuesA    | valuesB    |
|:-----------|:-----------|
| A          | B          |
| Fix\_1\_V2 | Fix\_1\_V2 |
| Fix\_2\_V3 | Fix\_2\_V3 |

We now have correct result (all cells filled).

`seplyr` used safe statement re-ordering to break the calculation into the minimum number of blocks/groups that have no in-block dependencies between statements (note this is more efficient that merely introducing a new mutate each first time a new value is used).

We can slow that down and see how the underlying planning functions break the assignments down into a small number of safe blocks (here we are using the development [`wrapr`](https://winvector.github.io/wrapr/) `1.0.2` function [`qae()`](https://winvector.github.io/wrapr/reference/qae.html)).

``` r
packageVersion("wrapr")
```

    ## [1] '1.0.2'

``` r
steps <- qae(valuesA := ifelse(is.na(valuesA) & canUseFix1, 
                               fix1, valuesA),
             valuesA := ifelse(is.na(valuesA) & canUseFix2, 
                               fix2, valuesA),
             valuesB := ifelse(is.na(valuesB) & canUseFix1, 
                               fix1, valuesB),
             valuesB := ifelse(is.na(valuesB) & canUseFix2, 
                               fix2, valuesB))
print(steps)
```

    ## $valuesA
    ## [1] "ifelse(is.na(valuesA) & canUseFix1, fix1, valuesA)"
    ## 
    ## $valuesA
    ## [1] "ifelse(is.na(valuesA) & canUseFix2, fix2, valuesA)"
    ## 
    ## $valuesB
    ## [1] "ifelse(is.na(valuesB) & canUseFix1, fix1, valuesB)"
    ## 
    ## $valuesB
    ## [1] "ifelse(is.na(valuesB) & canUseFix2, fix2, valuesB)"

``` r
plan <- partition_mutate_se(steps)
print(plan)
```

    ## $group00001
    ##                                              valuesA 
    ## "ifelse(is.na(valuesA) & canUseFix1, fix1, valuesA)" 
    ##                                              valuesB 
    ## "ifelse(is.na(valuesB) & canUseFix1, fix1, valuesB)" 
    ## 
    ## $group00002
    ##                                              valuesA 
    ## "ifelse(is.na(valuesA) & canUseFix2, fix2, valuesA)" 
    ##                                              valuesB 
    ## "ifelse(is.na(valuesB) & canUseFix2, fix2, valuesB)"

``` r
d %.>% 
  mutate_seb(., plan) %.>% 
  select_se(., c("valuesA", "valuesB")) %.>% 
  dplyr::collect(.) %.>% 
  knitr::kable(.)
```

| valuesA    | valuesB    |
|:-----------|:-----------|
| A          | B          |
| Fix\_1\_V2 | Fix\_1\_V2 |
| Fix\_2\_V3 | Fix\_2\_V3 |

For more on [`seplyr`](https://winvector.github.io/seplyr/) [please start here](http://winvector.github.io/FluidData/IntroductionToSeplyr.html).
