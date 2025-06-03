Data and Statistics in R - Aquatic Ecosystems
================
Jarrod Walton
2025-03-25

- [Data and Statistics in R - *Aquatic
  Ecosystems*](#data-and-statistics-in-r---aquatic-ecosystems)
  - [Introduction](#introduction)
- [1. Set up](#1-set-up)
  - [Loading libraries and data](#loading-libraries-and-data)
- [2. Visualisation](#2-visualisation)
  - [Save the plot](#save-the-plot)
- [3. Analysis](#3-analysis)
  - [ANOVA](#anova)
    - [Normal data - One-Way ANOVA](#normal-data---one-way-anova)
    - [Significant result - Post-hoc
      test](#significant-result---post-hoc-test)
    - [Non-normally distributed data - Kruskal-Wallis
      Test](#non-normally-distributed-data---kruskal-wallis-test)
    - [Post-hoc test - Dunn Test](#post-hoc-test---dunn-test)
- [Appendix](#appendix)
  - [What is Piping (`%>%`)](#what-is-piping-)
    - [How it works](#how-it-works)

# Data and Statistics in R - *Aquatic Ecosystems*

## Introduction

This is a tutorial for students who are either unable to use `PAST` or
are interested in improving their R skills.

The objectives are the same as those in the lab manual (*pp 44 - 51*)
and we will use the same dataset (Turbidity data).

The objectives are:

1.  Load libraries and get the data into R
2.  Construct a visualisation of the data, i.e, a barchart with SE bars.
3.  Check the data for normality using a **Shapiro-Wilk Tes**t
4.  Perform an analysis of variance (**ANOVA**) on the data to test
    whether there is a significant difference between sites. Depending
    on the distribution of the data, this will either be a standard
    parametric **One-Way ANOVA** or a non-parametric **Kruskal-Wallis
    Test**.
5.  If the results of the ANOVA are significant, perform a post-hoc test
    to determine which sites are different from each other. This will be
    a **Tukey’s HSD test** for parametric data and a **Dunn’s test**
    with a **Bonferroni adjustment** for non-parametric data.
6.  The write-up - *Reporting of the data is covered in the Lab Manual*

Packages used in this script will be `tidyverse`, `here`, and `FSA`. If
these packages are not installed, install them with:
install.packages(“package_name”), e.g., install.packages(“tidyverse”).

`tidyverse` is a collection of packages that work well together and are
designed for data science including `ggplot2`, `readr`, and many more.

`here` is completely optional. It is a package that makes it easier to
work with file paths. Instead of typing out the full path to a file, you
can use the `here` function to find the file in the working directory.

`FSA` is a package that contains functions for fisheries science and
aquatic ecology. It has heaps of functions including a nicely written
`dunnTest` function which is what we will be using it for.

*Note: There are many ways to do the same thing in R. This is just one
way to do it.*

# 1. Set up

The following code assumes you have the data file in the same directory
as this script. If not, you will need to change the path to the file.

Before running the following code. The data has been saved as a `.csv`
file and is called `aquaDataTurb.csv` (*See below*).

<img src="images/clipboard-4040950329.png" width="300" />

If you are not used to using scripts, you can try creating a new project
in the same place you’ve saved the csv file. This will automatically set
the working directory to the location of the project.

To do this, select File \> New Project \> Existing Directory (*See
Below*).

You can also check that you are currently in the correct working
directory by using the `getwd()` function. If you are not in the correct
directory, you can change it using the `setwd()` function.

<img src="images/File%20New%20Project.png" width="200" />

<img src="images/New%20Project%20Select.png" width="200" />

## Loading libraries and data

``` r
# Load required libraries
library(tidyverse)  # for data manipulation and plotting
library(here)       # for easier file paths (optional)
library(FSA)        # for Dunn's test and other ecology tools

# Set working directory using 'here' (ensure you're in an R project)
setwd(here())

# Load dataset
data <- read_csv(here("aquaDataTurb.csv"))
```

# 2. Visualisation

We will visualise the data using a barplot of the mean values per site
with standard error bars.

You may be unfamiliar with some parts of the code below. Don’t worry too
much about that for now, just try to understand the general structure of
the code.

We first take our dataframe (data) and pipe it (that’s this thing `%>%`,
*see Appendix* for more information) into ggplot with our aestheics
being Site on the x-axis and Turbidity on the y-axis.

We then add a geom_bar layer to plot the mean values of Turbidity per
site.

Next, we add a `geom_errorbar` layer to plot the standard error bars.

Finally, we add some theme elements to make the plot look nice and
customise the axis label so as to include the units.

To do this with other variables, you would simply change the x and y
aesthetics in the `ggplot` function and the label functions.

``` r
turb_plot <- data %>%
  ggplot(aes(x = Site, y = Turbidity)) +
  geom_bar(stat = "summary", fun = "mean", fill = "skyblue") +
  geom_errorbar(stat = "summary", fun.data = "mean_se", width = 0.2) +
  theme_classic() +
  theme(axis.text.x = element_text(angle = 45, hjust = 1)) +
  labs(x = NULL, y = "Turbidity (NTU)")

print(turb_plot)
```

![](Stats-in-R---Aquatic-Ecology_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

## Save the plot

Save the plot as a png if you like it.

Use the `ggsave` function to save the plot as a png file.

*Note: It will automatically use the last plot you created, and save it
to the working directory unless otherwise specified.*

You can change the size and resolution of the plot by adding arguments
to the ggsave function.

Check out the help file for more information. i.e., `?ggsave`

``` r
ggsave("Turbidity_plot.png")
```

# 3. Analysis

An ANOVA is a parametric test that assumes the data being analysed is
normally distributed. That is, the total error above and below the
population mean should add up to zero and the likelihood of sampling a
value becomes less likely the further from the mean it is.

Normality can be determined visually, but we will test it statistically
using a **Shapiro-Wilkes** test.

A significant result means the data is **not normally distributed** and
we will thus use a non-parametric test instead which does not assume
normally distributed data.

*Note: We want to know if the data within each site is normally
distributed, not the data as a whole.*

In the code below, we use pipes again which make the code more readable
(*See Appendix*).

We `group` the data by Site and then use the `summarise` function to
calculate the p-value of the Shapiro-Wilkes test for each site.

The `summarise` function applies a function to each level of what ever
you grouped the data by. In this case, it applies the `shapiro.test`
function to the Turbidity data for each site.

``` r
normality_results <- data %>%
  group_by(Site) %>%
  summarise(p_value = shapiro.test(Turbidity)$p.value)

print(normality_results)
```

    ## # A tibble: 5 × 2
    ##   Site   p_value
    ##   <chr>    <dbl>
    ## 1 Site 1  0.768 
    ## 2 Site 2  0.794 
    ## 3 Site 3  0.530 
    ## 4 Site 4  0.0859
    ## 5 Site 5  0.228

*Interperetation: If the p-value is less than 0.05, we reject the null
hypothesis that the data is normally distributed.*

## ANOVA

### Normal data - One-Way ANOVA

If the data is normally distributed, we can use a **One-Way ANOVA** to
test for differences between sites.

In R, a model is written with a `~` symbol where the variable on the
left is the response variable and those on the right of the `~` are the
independent variable(s).

Our research question is: “Is there a significant difference in
Turbidity between sites?” Thus,

*Turbidity ~ Site*

The model is then passed to the `aov` function along with the data.

To see the results, we pass the model to the `summary` function.

``` r
anova_turb <- aov(Turbidity ~ Site, data = data)

summary(anova_turb)
```

    ##             Df Sum Sq Mean Sq F value Pr(>F)    
    ## Site         4 1199.3   299.8   298.4 <2e-16 ***
    ## Residuals   30   30.1     1.0                   
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

### Significant result - Post-hoc test

If the ANOVA is significant, we can perform a **post-hoc test** to
determine which sites are different from each other.

In this case, we will use **Tukey’s HSD** test.

``` r
TukeyHSD(anova_turb)
```

    ##   Tukey multiple comparisons of means
    ##     95% family-wise confidence level
    ## 
    ## Fit: aov(formula = Turbidity ~ Site, data = data)
    ## 
    ## $Site
    ##                      diff         lwr          upr     p adj
    ## Site 2-Site 1 -12.9357143 -14.4899660 -11.38146254 0.0000000
    ## Site 3-Site 1  -1.4857143  -3.0399660   0.06853746 0.0663865
    ## Site 4-Site 1 -11.6728571 -13.2271089 -10.11860540 0.0000000
    ## Site 5-Site 1 -13.2100000 -14.7642517 -11.65574825 0.0000000
    ## Site 3-Site 2  11.4500000   9.8957483  13.00425175 0.0000000
    ## Site 4-Site 2   1.2628571  -0.2913946   2.81710889 0.1554000
    ## Site 5-Site 2  -0.2742857  -1.8285375   1.27996603 0.9855243
    ## Site 4-Site 3 -10.1871429 -11.7413946  -8.63289111 0.0000000
    ## Site 5-Site 3 -11.7242857 -13.2785375 -10.17003397 0.0000000
    ## Site 5-Site 4  -1.5371429  -3.0913946   0.01710889 0.0537125

### Non-normally distributed data - Kruskal-Wallis Test

If the data is not normally distributed, we can use a **Kruskal-Wallis**
test to test for differences between sites.

The code is very similar to the **ANOVA**, but we use the `kruskal.test`
function instead of the `aov` function.

We do not need to pass the results into the `summary` function as the
output is already in a readable format.

``` r
kruskal.test(Turbidity ~ Site, data = data)
```

    ## 
    ##  Kruskal-Wallis rank sum test
    ## 
    ## data:  Turbidity by Site
    ## Kruskal-Wallis chi-squared = 30.273, df = 4, p-value = 4.307e-06

### Post-hoc test - Dunn Test

If the **Kruskal-Wallis** test was significant, we can perform a
post-hoc test to determine which sites are different from each other.

The **Dunn test** is a non-parametric equivalent to Tukey’s HSD test.

Because the data was not normally distributed, we use a **Bonferroni
adjustment** to account for the pitfalls of multiple comparisons. i.e.,
we want to avoid making a **Type I error**.

The syntax is similar to the previous models.

*Remember to refer to the adjusted p-values when interpreting the
results.*

``` r
dunnTest(Turbidity ~ Site, data = data, method = "bonferroni")
```

    ##         Comparison          Z      P.unadj        P.adj
    ## 1  Site 1 - Site 2  3.8614922 1.126966e-04 1.126966e-03
    ## 2  Site 1 - Site 3  0.6783703 4.975370e-01 1.000000e+00
    ## 3  Site 2 - Site 3 -3.1831220 1.456962e-03 1.456962e-02
    ## 4  Site 1 - Site 4  2.2568857 2.401522e-02 2.401522e-01
    ## 5  Site 2 - Site 4 -1.6046066 1.085804e-01 1.000000e+00
    ## 6  Site 3 - Site 4  1.5785154 1.144473e-01 1.000000e+00
    ## 7  Site 1 - Site 5  4.4876802 7.200292e-06 7.200292e-05
    ## 8  Site 2 - Site 5  0.6261879 5.311917e-01 1.000000e+00
    ## 9  Site 3 - Site 5  3.8093099 1.393552e-04 1.393552e-03
    ## 10 Site 4 - Site 5  2.2307945 2.569474e-02 2.569474e-01

# Appendix

## What is Piping (`%>%`)

*“Piping”* is used preferentially as an aesthetic choice. It enhances
readability.

### How it works

It uses the pipe operator (`%>%`) which takes whatever is fed into it
(e.g., `Data`) and passes it into the next function as the first
argument.

That is,

`x %>% function(df)`

is the same as:

`function(x, df)`

In other words:

`"I will take an apple, slice it, then eat it"`

is the same as:

`"I will eat a sliced apple"`

Or, if translating to R:

`Apple %>% Slice() %>% Eat()`

is the same as:

`Eat(Slice(Apple))`

The first “sentences” in the above examples may seem more verbose, but
if aiming to have your code understandable rather than purely
functional, then piping is preferred.

This is especially the case when the functions take multiple arguments.
Suppose we want to specify what we will slice the apple with (“knife”):
`Slice(Apple, with = "knife")`

Then, the piping version is:

e.g., `Apple %>% Slice(with = "knife") %>% Eat()`

which is more readable from left to right than:

`Eat(Slice(Apple, with = "knife"))`
