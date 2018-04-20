<!-- README.md is generated from README.Rmd. Please edit that file -->

[![Travis build
status](https://travis-ci.org/ColinFay/attempt.svg?branch=master)](https://travis-ci.org/ColinFay/attempt)
[![Coverage
status](https://codecov.io/gh/ColinFay/attempt/branch/master/graph/badge.svg)](https://codecov.io/github/ColinFay/attempt?branch=master)
[![AppVeyor Build
Status](https://ci.appveyor.com/api/projects/status/github/ColinFay/attempt?branch=master&svg=true)](https://ci.appveyor.com/project/ColinFay/attempt)
[![](https://cranlogs.r-pkg.org/badges/attempt)](https://cran.rstudio.com/web/packages/attempt/index.html)

# {attempt} 0.2.1

A Friendlier Condition Handler for R, inspired by {purrr} mappers and
based on {rlang}.

{attempt} is designed to handle the cases when something / someone
attempts to do something it shouldn’t.

For example :

  - an attempt to run a `log("a")` (error)
  - an attempt to connect to a web API without an internet connexion
    (error)
  - an attempt to `paste()` `"good morning"` and `iris`
    (message/warning)
  - …

{attempt} provides several condition handlers, from try catch to simple
message printing.

{attempt} only depends on {rlang}, and every function is design to be
fast, making it easy to implement in other functions and packages.

# Install

From CRAN:

``` r
install.packages("attempt")
```

The dev version:

``` r
devtools::install_github("ColinFay/attempt")
```

# Reference

``` r
library(attempt)
```

## attempt

`attempt` is a wrapper around base `try` that allows you to insert a
custom messsage on error.

``` r
attempt(log("a"))
# Error: argument non numérique pour une fonction mathématique
attempt(log("a"), msg = "Nop !")
# Error: Nop !
```

You can make it verbose (i.e. returning the expression):

``` r
attempt(log("a"), msg = "Nop !", verbose = TRUE)
# Error in log("a"): Nop !
```

Of course the result is returned if there is one:

``` r
attempt(log(1), msg = "Nop !", verbose = TRUE)
# [1] 0
```

As with `try`, the result cant be saved as an error object:

``` r
a <- attempt(log("a"), msg = "Nop !", verbose = TRUE)
a
# [1] "Error in log(\"a\"): Nop !\n"
# attr(,"class")
# [1] "try-error"
# attr(,"condition")
# <simpleError in log("a"): Nop !>
```

## silent\_attempt

`silent_attempt` is a wrapper around `silently` (see further down for
more info) and `attempt`. It attempts to run the expr, stays silent if
the expression succeeds, and returns error or warnings if any.

``` r
silent_attempt(log("a"))
# Error: argument non numérique pour une fonction mathématique
silent_attempt(log(1))
```

## try catch

You can write a try catch with these params :

  - `expr` the expression to be evaluated
  - `.e` a mapper or a function evaluated when an error occurs
  - `.w` a mapper or a function evaluated when a warning occurs
  - `.f` a mapper or an expression which is always evaluated before
    returning or exiting

In `.e` and `.f`, the `.x` refers to the error / warning object.

### With mappers

``` r
try_catch(expr = log("a"), 
          .e = ~ paste0("There is an error: ", .x), 
          .w = ~ paste0("This is a warning: ", .x))
#[1] "There is an error: Error in log(\"a\"): argument non numérique pour une fonction mathématique\n"

try_catch(log("a"), 
          .e = ~ stop(.x), 
          .w = ~ warning(.x))
# Error in log("a") : argument non numérique pour une fonction mathématique

try_catch(matrix(1:3, nrow= 2), 
          .e = ~ print(.x), 
          .w = ~ print(.x))
#<simpleWarning in matrix(1:3, nrow = 2): la longueur des données [3] n'est pas un diviseur ni un multiple du nombre de lignes [2]>

try_catch(expr = 2 + 2 , 
          .f = ~ print("Using R for addition... ok I'm out!"))
# [1] "Using R for addition... ok I'm out!"
# [1] 4
```

As usual, the handlers are set only if you call them :

``` r
try_catch(matrix(1:3, nrow = 2), .e = ~ print("error"))
#      [,1] [,2]
# [1,]    1    3
# [2,]    2    1
# Warning message:
# In matrix(1:3, nrow = 2) :
#   la longueur des données [3] n'est pas un diviseur ni un multiple du nombre de lignes [2]
```

``` r
try_catch(matrix(1:3, nrow = 2), .w = ~ print("warning"))
# [1] "warning"
```

### Traditionnal way

{attempt} is flexible in how you can specify your arguments.

You can, as you do with {base} `tryCatch`, use a plain old function:

``` r
try_catch(log("a"), 
          .e = function(e){
            print(paste0("There is an error: ", e))
            print("Ok, let's save this")
            time <- Sys.time()
            a <- paste("+ At",time, ", \nError:",e)
            # write(a, "log.txt", append = TRUE) # commented to prevent log.txt creation 
            print(paste("log saved on log.txt at", time))
            print("let's move on now")
          })

# [1] "There is an error: Error in log(\"a\"): argument non numérique pour une fonction mathématique\n"
# [1] "Ok, let's save this"
# [1] "log saved on log.txt at 2018-01-30 16:59:13"
# [1] "let's move on now"
```

You can even mix both:

``` r
try_catch(log("a"), 
          .e = function(e){
            paste0("There is an error: ", e)
          },
          .f = ~ print("I'm not sure you can do that pal !"))
# [1] "I'm not sure you can do that pal !"
# [1] "There is an error: Error in log(\"a\"): argument non numérique pour une fonction mathématique\n"

try_catch(log("a"), 
          .e = ~ paste0("There is an error: ", .x),
          .f = function() print("I'm not sure you can do that pal !"))
# [1] "I'm not sure you can do that pal !"
# [1] "There is an error: Error in log(\"a\"): argument non numérique pour une fonction mathématique\n"
```

### try\_catch\_df

`try_catch_df` returns a tibble with the call, the error message if any,
the warning message if any, and the value of the evaluated expression or
“error”. The values will always be contained in a list-column.

``` r
res_log <- try_catch_df(log("a"))
res_log
#>       call                                                 error warning
#> 1 log("a") argument non numérique pour une fonction mathématique      NA
#>   value
#> 1 error
res_log$value
#> [[1]]
#> [1] "error"

res_matrix <- try_catch_df(matrix(1:3, nrow = 2))
res_matrix
#>                    call error
#> 1 matrix(1:3, nrow = 2)    NA
#>                                                                                    warning
#> 1 la longueur des données [3] n'est pas un diviseur ni un multiple du nombre de lignes [2]
#>        value
#> 1 1, 2, 3, 1
res_matrix$value
#> [[1]]
#>      [,1] [,2]
#> [1,]    1    3
#> [2,]    2    1

res_success <- try_catch_df(log(1))
res_success
#>     call error warning value
#> 1 log(1)    NA      NA     0
res_success$value
#> [[1]]
#> [1] 0
```

### map try\_catch

`map_try_catch` and `map_try_catch_df` allow you to map on a list of
arguments `l`, to be evaluated by the function in `fun`.

``` r
map_try_catch(l = list(1, 3, "a"), fun = log, .e = ~ .x)
#> [[1]]
#> [1] 0
#> 
#> [[2]]
#> [1] 1.098612
#> 
#> [[3]]
#> <simpleError in .Primitive("log")("a"): argument non numérique pour une fonction mathématique>

map_try_catch_df(list(1,3,"a"), log)
#>                     call
#> 1   .Primitive("log")(1)
#> 2   .Primitive("log")(3)
#> 3 .Primitive("log")("a")
#>                                                   error warning    value
#> 1                                                  <NA>      NA        0
#> 2                                                  <NA>      NA 1.098612
#> 3 argument non numérique pour une fonction mathématique      NA    error
```

## Adverbs

Adverbs take a function and return a modified function.

### silently

`silently` transforms a function so that when you call this new
function, it returns nothing unless there is an error or a warning
(contrary to `attempt` that returns the result). In a sense, the new
function stay silent unless error or warning.

``` r
silent_log <- silently(log)
silent_log(1)
silent_log("a")
# Error in .f(...) : argument non numérique pour une fonction mathématique
```

With `silently`, the result is never returned.

``` r
silent_matrix <- silently(matrix)
silent_matrix(1:3, 2)
#Warning message:
#In .f(...) :
#  la longueur des données [3] n'est pas un diviseur ni un multiple du nombre de lignes [2]
```

### surely

`surely` transforms a function so that when you call this new function,
it calls `attempt()` - i.e. in the code below, calling `sure_log(1)` is
the same as calling `attempt(log(1))`. In a sense, you’re sure this new
function will always work.

``` r
sure_log <- surely(log)
sure_log(1)
# [1] 0
sure_log("a")
# Error: argument non numérique pour une fonction mathématique
```

### `with_message` and `with_warning`

These two functions take a function, and add a warning or a message to
it.

``` r
as_num_msg <- with_message(as.numeric, msg = "We're performing a numeric conversion")
as_num_warn <- with_warning(as.numeric, msg = "We're performing a numeric conversion")
as_num_msg("1")
#> We're performing a numeric conversion
#> [1] 1
as_num_warn("1")
#> Warning in as_num_warn("1"): We're performing a numeric conversion
#> [1] 1
```

### `without_message` and `withou_warning`

These two functions do the opposite, as they remove warnings and
messages:

``` r
matrix(1:3, ncol = 2)
no_warning_matrix <- without_warning(matrix)
no_warning_matrix(1:3, ncol = 2)
```

## `if_` conditions

`if_none`, `if_any` and `if_all` test the elements of the list.

``` r
if_all(1:10, ~ .x < 11, ~ return(letters[1:10]))
#>  [1] "a" "b" "c" "d" "e" "f" "g" "h" "i" "j"

if_any(1:10, is.numeric, ~ "Yay!")
#> [1] "Yay!"

if_none(1:10, is.character, ~ rnorm(10))
#>  [1] -1.354288865 -1.049826692 -1.378733301  0.005942126 -1.380438418
#>  [6]  0.353616730 -0.489985293 -0.019115143 -2.262219617 -0.295913462
```

The defaut for all `.p` is `isTRUE`. So you can:

``` r
a <- c(FALSE, TRUE, TRUE, TRUE)

if_any(a, .f = ~ "nop!")
#> [1] "nop!"
```

`if_then` performs a simple “if this then do that”:

``` r
if_then(1, is.numeric, ~ "nop!")
#> [1] "nop!"
```

`if_not` runs `.f` if `.p(.x)` is not TRUE :

``` r
if_not(.x = 1, .p = is.character, ~ ".x is not a character")
#> [1] ".x is not a character"
```

And `if_else` is a wrapper around `base::ifelse()`.

If you want these function to return a value, you need to wrap these
values into a mapper / a function. E.g, to return a vector, you’ll need
to write `if_then(1, is.numeric, ~ "Yay")`.

``` r
a <- if_else(1, is.numeric, ~ "Yay", ~ "Nay")
a
#> [1] "Yay"
```

## warnings and messages

The `stop_if`, `warn_if` and `message_if` are easy to use functions that
send an error, a warning or a message if a condition is met. Each
function has its counterpart with `_not` that returns a message if the
condition is not met.

`stop_if_not` is quite the same as `assert_that` from the {assertthat}
package, except that it can takes mappers. It is not the same as base
`stopifnot()`, as it doesn’t take a list of expression.

These functions are also flexible as you can pass base predicates
(is.numeric, is.character…), a custom predicate built with mappers, or
even your own predicate function.

You can either choose a custom message or just let the built-in messages
be printed:

``` r
x <- 12
# Stop if .x is numeric
stop_if(.x = x, 
        .p = is.numeric)
#> Error: Test `is.numeric` on `x` returned an error.

y <- "20"
# stop if .x is not numeric
stop_if_not(.x = y, 
            .p = is.numeric, 
            msg = "y should be numeric")
#> Error: y should be numeric
a  <- "this is not numeric"
# Warn if .x is charcter
warn_if(.x = a, 
        .p = is.character)
#> Warning: Test `is.character` on `a` returned a warning.

b  <- 20
# Warn if .x is not equal to 10
warn_if_not(.x = b, 
        .p = ~ .x == 10 , 
        msg = "b should be 10")
#> Warning: b should be 10

c <- "a"
# Message if c is a character
message_if(.x = c, 
           .p = is.character, 
           msg = "You entered a character element")
#> You entered a character element

# Build more complex predicates
d <- 100
message_if(.x = d, 
           .p = ~ sqrt(.x) < 42, 
           msg = "The square root of your element must be more than 42")
#> The square root of your element must be more than 42

# Or, if you're kind of old school, you can still pass classic functions

e <- 30
message_if(.x = e, 
           .p = function(vec){
             return(sqrt(vec) < 42)
           }, 
           msg = "The square root of your element must be more than 42")
#> The square root of your element must be more than 42
```

If you need to call a function that takes no argument at `.p` (like
`curl::has_internet()`), use this function as
`.x`.

``` r
stop_if(.x = curl::has_internet(), msg = "You shouldn't have internet to do that")
#> Error: You shouldn't have internet to do that

warn_if(.x = curl::has_internet(), 
            msg = "You shouldn't have internet to do that")
#> Warning: You shouldn't have internet to do that

message_if(.x = curl::has_internet(), 
            msg = "Huray, you have internet \\o/")
#> Huray, you have internet \o/
```

If you don’t specify a `.p`, the default test is `isTRUE`.

``` r
a <- is.na(airquality$Ozone)
message_if_any(a, msg = "NA found")
#> NA found
```

### In function

That can come really handy inside a function :

``` r
my_fun <- function(x){
  stop_if_not(.x = curl::has_internet(), 
              msg = "You should have internet to do that")
  warn_if_not(x, 
          is.character, 
          msg =  "x is not a character vector. The output may not be what you're expecting.")
  paste(x, "is the value.")
}

my_fun(head(iris))
#> Warning: x is not a character vector. The output may not be what you're
#> expecting.
#> [1] "c(5.1, 4.9, 4.7, 4.6, 5, 5.4) is the value."  
#> [2] "c(3.5, 3, 3.2, 3.1, 3.6, 3.9) is the value."  
#> [3] "c(1.4, 1.4, 1.3, 1.5, 1.4, 1.7) is the value."
#> [4] "c(0.2, 0.2, 0.2, 0.2, 0.2, 0.4) is the value."
#> [5] "c(1, 1, 1, 1, 1, 1) is the value."
```

### none, all, any

`stop_if`, `warn_if` and `message_if` all have complementary tests with
`_all`, `_any` and `_none`, which combine the `if_*` and the `warn_*`,
`stop_*` and `message_*` seen before. They take a list as first
argument, and a predicate. They test if any, all or none of the elements
validate the
predicate.

``` r
stop_if_any(iris, is.factor, msg = "Factors here. This might be due to stringsAsFactors.")
#> Error: Factors here. This might be due to stringsAsFactors.

warn_if_none(1:10, ~ .x < 0, msg = "You need to have at least one number under zero.")
#> Warning: You need to have at least one number under zero.

message_if_all(1:100, is.numeric, msg = "That makes a lot of numbers.")
#> That makes a lot of numbers.
```

# Misc

## Acknowledgments

Thanks to [Romain](http://romain.rbind.io/) for the name suggestion.

## Contact

Questions and feedbacks [welcome](mailto:contact@colinfay.me)\!

You want to contribute ? Open a
[PR](https://github.com/ColinFay/attempt/pulls) :) If you encounter a
bug or want to suggest an enhancement, please [open an
issue](https://github.com/ColinFay/attempt/issues).

Please note that this project is released with a [Contributor Code of
Conduct](CONDUCT.md). By participating in this project you agree to
abide by its terms.
