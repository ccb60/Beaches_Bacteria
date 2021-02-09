Analysis of Bacteria Levels at Casco Bay Beaches
================
Curtis C. Bohlen, Casco Bay Estuary Partnership.
01/23/2021

-   [Introduction](#introduction)
    -   [Standards](#standards)
        -   [Beaces Program](#beaces-program)
        -   [Maine State Class SB Waters
            Standards](#maine-state-class-sb-waters-standards)
-   [Import Libraries](#import-libraries)
-   [Data Preparation](#data-preparation)
    -   [Initial Folder References](#initial-folder-references)
    -   [Load Data](#load-data)
    -   [Add a “Beach” Identifier](#add-a-beach-identifier)
    -   [Add a “Day of the Week”
        Identifier](#add-a-day-of-the-week-identifier)
    -   [Add Maximum Likelihood estimate for
        non-detects](#add-maximum-likelihood-estimate-for-non-detects)
    -   [Calculate Exceedences](#calculate-exceedences)
-   [Recent Status](#recent-status)
    -   [Graphic Concepts](#graphic-concepts)
    -   [Create Geometric Mean
        Function](#create-geometric-mean-function)
    -   [Dot Plot (Fails….)](#dot-plot-fails.)
    -   [Jitter Plot Bare](#jitter-plot-bare)
        -   [Add Geometric Means](#add-geometric-means)
    -   [Violin Plot with Jitter](#violin-plot-with-jitter)
        -   [Add Geometric Means](#add-geometric-means-1)
    -   [Boxplot with Jitter](#boxplot-with-jitter)
        -   [With Color by Rainfall](#with-color-by-rainfall)
        -   [With Color by Detection](#with-color-by-detection)
        -   [Add Annotation](#add-annotation)
        -   [Add Geometric Means](#add-geometric-means-2)
-   [Trend Graphics](#trend-graphics)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

To be added….

## Standards

### Beaces Program

104 CFU / 100 ml, for individual observations.

### Maine State Class SB Waters Standards

> the number of enterococcus bacteria in these waters may not exceed a
> geometric mean of 8 CFU per 100 milliliters in any 90-day interval or
> 54 CFU per 100 milliliters in more than 10% of the samples in any
> 90-day interval.

38 M.R.S. §465-B(2)(B)

# Import Libraries

``` r
library(fitdistrplus)
#> Loading required package: MASS
#> Loading required package: survival
library(tidyverse)
#> -- Attaching packages --------------------------------------- tidyverse 1.3.0 --
#> v ggplot2 3.3.3     v purrr   0.3.4
#> v tibble  3.0.5     v dplyr   1.0.3
#> v tidyr   1.1.2     v stringr 1.4.0
#> v readr   1.4.0     v forcats 0.5.0
#> -- Conflicts ------------------------------------------ tidyverse_conflicts() --
#> x dplyr::filter() masks stats::filter()
#> x dplyr::lag()    masks stats::lag()
#> x dplyr::select() masks MASS::select()

# library(GGally)

library(mgcv)      # For GAMs and GAMMs; used here for seasonal smoothers
#> Loading required package: nlme
#> 
#> Attaching package: 'nlme'
#> The following object is masked from 'package:dplyr':
#> 
#>     collapse
#> This is mgcv 1.8-33. For overview type 'help("mgcv-package")'.
library(emmeans)   # For marginal means

library(mblm)      # for the Thiel-Sen estimators

library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())

library(LCensMeans)
```

# Data Preparation

## Initial Folder References

``` r
sibfldnm    <- 'Derived_Data'
parent      <- dirname(getwd())
sibling     <- file.path(parent,sibfldnm)

dir.create(file.path(getwd(), 'figures'), showWarnings = FALSE)
#dir.create(file.path(getwd(), 'models'),  showWarnings = FALSE)
```

## Load Data

``` r
fn <- "beaches_data.csv"
beach_data <- read_csv(file.path(sibling, fn))
#> 
#> -- Column specification --------------------------------------------------------
#> cols(
#>   .default = col_double(),
#>   SiteCode = col_character(),
#>   sdatetime = col_datetime(format = ""),
#>   sdate = col_date(format = ""),
#>   Sample_ID = col_character(),
#>   Sample_Qualifier = col_character(),
#>   Lab_Qualifier = col_character(),
#>   Censored_Flag = col_logical(),
#>   Weather = col_character(),
#>   Past24HR_Weather = col_character(),
#>   Past48HR_Weather = col_character(),
#>   Tide_Stage = col_logical(),
#>   Water_Surface = col_character(),
#>   Current = col_logical()
#> )
#> i Use `spec()` for the full column specifications.
```

## Add a “Beach” Identifier

``` r
fn = "beach_locations.csv"
beach_lookup = read_csv(file.path(sibling, fn),
                        col_types = cols(
                          Town = col_character(),
                          Beach_Name = col_character(),
                          SamplePoint = col_character(),
                          Latitude = col_double(),
                          Longitude = col_double()
                        )) %>%
  select(-Latitude, -Longitude)

beach_data <- beach_data %>%
  mutate(Beach = beach_lookup$Beach_Name[match(SiteCode, 
                                               beach_lookup$SamplePoint)])
```

## Add a “Day of the Week” Identifier

We need this to help evaluate whether samples are “normal” samples or
“storm” samples.

``` r
beach_data <- beach_data %>%
  mutate(Weekday = weekdays(sdate)) %>%
  relocate(Weekday, .after = Month)
```

## Add Maximum Likelihood estimate for non-detects

This uses our LCensMeans package, and estimates a maximum likelihood
estimate of the expected value of the (unobserved) left censored values.
It relies on several assumption that are questionable for these data,
but it is arguably better than using the detection limit or half the
detection limit.

``` r
beach_data <- beach_data %>%
mutate(Bacteria2 = sub_cmeans(Bacteria, Censored_Flag) )
```

## Calculate Exceedences

``` r
beach_data <- beach_data %>%
  mutate(Exceeds = Bacteria > 104) %>%
  relocate(Exceeds, .after = Censored_Flag)
```

# Recent Status

``` r
recent_data <- beach_data %>%
  filter(Year > 2015)

recent_data %>%
  group_by(SiteCode) %>%
  summarize( years = length(unique(Year)),
             median_Bacteria = median(Bacteria2, na.rm = TRUE),
             gmean_bacteria = exp(mean(log(Bacteria2),nas.rm = TRUE)),
             mean_Bacteria = mean(Bacteria2, na.rm = TRUE),
             n = sum(! is.na(Bacteria)),
             n_exceeds = sum(Exceeds, na.rm = TRUE),
             p_exceeds = n_exceeds / n)
#> # A tibble: 6 x 8
#>   SiteCode years median_Bacteria gmean_bacteria mean_Bacteria     n n_exceeds
#> * <chr>    <int>           <dbl>          <dbl>         <dbl> <int>     <int>
#> 1 BC-1         4            3.54           7.44         37.1     51         2
#> 2 EEB-01       4           10              9.43         36.1    103         8
#> 3 HARP-1       2            3.41           3.88          4.20    25         0
#> 4 HARP-2       2            3.54           7.38         18.4     26         1
#> 5 HARP-3       2           20             17.1          41.9     26         4
#> 6 WIL-02       4           10             14.9         244.     105         9
#> # ... with 1 more variable: p_exceeds <dbl>
```

Note that the median BActeria for pretty much all of these stations is
at or below the lower detection limit. (That is possible because we have
replaced non-detects by an estimate of the conditional mean expected for
unobserved censored values). That means the data is at or below the
detection limits more than 50% of the time.

``` r
cat('Omitting non-detects\n')
#> Omitting non-detects
summary(recent_data$Enterococci)
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max.    NA's 
#>      10      10      20     189      52   15531     165
cat('\nNon-detects at Detection Limit\n')
#> 
#> Non-detects at Detection Limit
summary(recent_data$Bacteria)
#>    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
#>    10.0    10.0    10.0   101.1    20.0 15531.0
cat('\nNon-detects at maximum likelihood estimator\n')
#> 
#> Non-detects at maximum likelihood estimator
summary(recent_data$Bacteria2)
#>      Min.   1st Qu.    Median      Mean   3rd Qu.      Max. 
#>     3.205     3.441    10.000    97.864    20.000 15531.000
cat('\nGeometric Mean\n')
#> 
#> Geometric Mean
exp(mean(log(recent_data$Bacteria2)))
#> [1] 10.08517
cat('\n\nProbability of Violating Standards\n')
#> 
#> 
#> Probability of Violating Standards
sum(recent_data$Exceeds) / sum(! is.na(recent_data$Exceeds))
#> [1] 0.07142857
```

## Graphic Concepts

## Create Geometric Mean Function

``` r
gm_mean <- function(x) {
  exp(mean(log(x)))
}
```

## Dot Plot (Fails….)

``` r
recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria2, fill = Beach)) +
  geom_dotplot(binaxis = "y", 
               stackdir = "centerwhole", 
               position = "dodge",
               binpositions = 'all',
               method = 'histodot', 
               shape = 22,
               stroke = 0,
               binwidth = .2) +
  stat_summary(fun = gm_mean, fill = 'red', shape = 23) +
  
  geom_hline(yintercept = 104, color = 'gray50', lty = 3) +
  
  scale_y_log10() +
  scale_fill_manual(values = cbep_colors()) +

    theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1),
        legend.position = 'none') +

  ylab('Enterococci (MPN)') +
  xlab('') 
#> Warning: Ignoring unknown parameters: shape
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/dot_plot-1.png" style="display: block; margin: auto;" />

``` r
  #geom_bar(data = plain_emms, mapping = aes(Beach, response))
```

## Jitter Plot Bare

``` r
jitter_plt <- recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria2)) +
  
  geom_jitter(aes(color = Censored_Flag),
              width = 0.2, 
              height = .05,
              alpha = .75) +
  
  scale_y_log10() +
  scale_color_manual(values = cbep_colors(), 
                     name = '', labels = c('Observed', 'Below Detection')) +
  
  theme_cbep(base_size = 12) +

  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +
  theme(legend.position = c(.65, .9)) +

  ylab('Enterococci (MPN)') +
  xlab('')
jitter_plt
```

<img src="beaches_graphics_files/figure-gfm/jitter_bare-1.png" style="display: block; margin: auto;" />

### Add Geometric Means

This looks like the geometric means annotation are not properly lined
up, but it is lined up better in the PDF version, which is what counts.

``` r
xanchor <- 3.75
yanchor <- 2200

jitter_plt <- jitter_plt + 
  stat_summary(fun = gm_mean, fill = 'gray85', shape = 23) +
  
  annotate('point', x= xanchor, y = yanchor,
            size = 3, pch = 23, fill = 'gray85') +

  annotate('text', x= xanchor + 0.25, y = yanchor,
           hjust = 0, size = 3.5, label = 'Geometric Mean')
jitter_plt
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/jitter_w_geom_mean-1.png" style="display: block; margin: auto;" />
\#\#\# Add Annotations

``` r
jitter_plt +
  geom_hline(yintercept = 104, color = 'gray50', lty = 3) #+
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/jitter_all-1.png" style="display: block; margin: auto;" />

``` r
  #geom_hline(yintercept = 8, color = 'gray50', lty = 3) +
  
  #annotate('text', x = 0, y = 80, label = 'Acute = 104', size = 2.5, hjust = 0) +
  #annotate('text', x = 0, y = 6, label = 'Chronic = 8', size = 2.5, hjust = 0) 


ggsave('figures/recent_conditons_jitter.pdf', device = cairo_pdf, 
       width = 5, height = 4)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

## Violin Plot with Jitter

The challenge here is that if we make width proportional to number of
observations, the large number of observations and non-detect at
Stover’s Point dominates. So we make all sites equal width , and show
dots to signal sample size.

``` r
violin_plt <- recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria2)) +
  geom_violin(scale = 'width',
              width = .75,
              # draw_quantiles = c(0.25, 0.5, 0.75),
              fill = cbep_colors()[6],
              color = 'gray75',
              alpha = 0.1) +
  geom_jitter(aes(color = Censored_Flag),
              width = 0.15, 
              height = .05,
              alpha = 0.75) +
  
  geom_hline(yintercept = 104, color = 'gray50', lty = 3) +
 # geom_hline(yintercept = 8, color = 'gray50', lty = 3) +
  
  scale_y_log10() +
  scale_color_manual(values = cbep_colors(), 
                     name = '', labels = c('Observed', 'Below Detection')) +
  
  theme_cbep(base_size = 12) +

  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +
  theme(legend.position = c(.65, .9)) +

  ylab('Enterococci (MPN)') +
  xlab('')
violin_plt
```

<img src="beaches_graphics_files/figure-gfm/violin_start-1.png" style="display: block; margin: auto;" />

### Add Geometric Means

``` r
xanchor <- 3.5
yanchor <- 1500

violin_plt + 
  stat_summary(fun = gm_mean, fill = 'gray85', shape = 23) +
  
  annotate('point', x= xanchor, y = yanchor,
            size = 3, pch = 23, fill = 'gray85') +

  annotate('text', x= xanchor + 0.3, y = yanchor,
           hjust = 0, size = 3.5, label = 'Geometric Mean')
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/violin_final-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/recent_conditons_jitter_violin.pdf', device = cairo_pdf, 
       width = 5, height = 4)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

## Boxplot with Jitter

### With Color by Rainfall

``` r
recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria2)) +
  geom_boxplot(width = .6, outlier.shape = NA) +
  geom_jitter(aes(color = Rain48), width = 0.25, 
              height = .05, 
              alpha = 1) +
  
  geom_hline(yintercept = 104, color = 'gray50', lty = 3) +
  #geom_hline(yintercept = 8, color = 'gray50', lty = 3) +
  
  scale_y_log10() +
  
  scale_color_viridis_c('Inches of Rain\n Prior 48 Hours') +
  
  theme_cbep(base_size = 12) +

  theme(legend.key.size = unit(10, 'points'),
        legend.title = element_text(size = 9)) +
  
  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +

  ylab('Enterococci (MPN)') +
  xlab('') +
  stat_summary(fun = gm_mean, fill = 'gray85', shape = 23)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/boxplot_rainfall-1.png" style="display: block; margin: auto;" />

### With Color by Detection

``` r
plt <- recent_data %>%
  ggplot(aes(x = Beach, y = Bacteria2)) +
  geom_boxplot(width = .6, outlier.shape = NA) +
  geom_jitter(aes(color = Censored_Flag),
              width = 0.25, 
              height = .05, 
              #color = cbep_colors()[5],
              alpha = 0.75) +
  
  geom_hline(yintercept = 104, color = 'gray50', lty = 3) +
  #geom_hline(yintercept = 8, color = 'gray50', lty = 3) +
  
  scale_y_log10() +
  
  scale_color_manual(values = cbep_colors(), 
                     labels = c('Observed', 'Below Detection'),
                     name = NULL) +
  
  theme_cbep(base_size = 12) +

  theme(axis.text.x = element_text(angle = 45, size = 9, hjust = 1)) +
  theme(legend.position = c(.65, .875),
        legend.text = element_text(size = 8),
        legend.key.height = unit(10, 'points')) +

  ylab('Enterococci (MPN)') +
  xlab('')

plt +
  stat_summary(fun = gm_mean, fill = 'gray85', shape = 23)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/boxplot_detection-1.png" style="display: block; margin: auto;" />

### Add Annotation

``` r
# Calculate positions for annotations
xanchor <- 0.75

# On a log scale, equally spaced items have the same RATIO
yanchor <- 6000
ylow    <-  2/3*yanchor
yhigh    <-  3/2* yanchor

plt2 <- plt + 
  annotate('rect', xmin = xanchor, ymin = ylow, xmax =xanchor + 0.1, ymax = yhigh,
           fill = 'white', color = 'gray30', size = .5) + 
  annotate('segment', x= xanchor, y = yanchor, xend = xanchor + 0.1, yend = yanchor, 
           color = 'gray30') +

  annotate('text', x= xanchor + 0.2, y = ylow,
           hjust = 0, size = 3, label = '25th percentile') +
  annotate('text', x= xanchor + 0.2, y = yanchor,
           hjust = 0, size = 3, label = 'median') +
  annotate('text', x= xanchor + 0.2, y = yhigh,
           hjust = 0, size = 3, label = '75th percentile')
  
  
plt2
```

<img src="beaches_graphics_files/figure-gfm/boxplot_annotation-1.png" style="display: block; margin: auto;" />

### Add Geometric Means

``` r
plt2 + 
   stat_summary(fun = gm_mean, shape = 23, fill = 'gray85') +

annotate('point', x= xanchor, y = yanchor * (4/9),
            size = 3, pch = 23, fill = 'gray85') +

annotate('text', x= xanchor + 0.2, y = yanchor * (4/9),
           hjust = 0, size = 3, label = 'geometric mean')
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

<img src="beaches_graphics_files/figure-gfm/boxplot_final-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/recent_conditons_jitter_box.pdf', device = cairo_pdf, 
       width = 5, height = 4)
#> Warning: Removed 6 rows containing missing values (geom_segment).
```

# Trend Graphics

We only have long term data from two beaches – East End and Willard. And
trends are not statistically significant at eh single beach level.
Apparent importance depends on the model uses, which is always
uncomfortable.

For simplicity, we restrict attention to those to near-continuous
records only.

``` r
trend_data <- beach_data %>%
  filter(SiteCode == 'WIL-02' | SiteCode == 'EEB-01') %>%
  filter(! is.na(Bacteria2)) %>%
  filter(Year > 2004) %>%
  filter(Reporting_Limit > 5 | is.na(Reporting_Limit))
```

``` r
plt <- ggplot(trend_data, aes(x = Year, y = Bacteria2, color = Censored_Flag)) +
  geom_jitter(alpha = .5, height = 0.025, width = 0.2) +
  scale_y_log10() +
  scale_x_continuous(breaks = c(2006, 2010, 2014, 2018)) +
  
  xlab('') +
  ylab('Enterococci (MPN)') +
  
  theme_cbep(base_size = 14) +
  theme(panel.border = element_rect(color = 'gray85', fill = NA)) + 

  scale_color_manual(values = cbep_colors(), 
                     labels = c('Observed', 'Below Detection'),
                     name = NULL,
                     guide = guide_legend(ncol  = 2)) +
  # Following adds a little extra space above the plot to make room
  # for a legend, and places the legend a bit closer to the plot
  # than the default "top" position.
  
  theme( plot.margin=unit(c(.25,.25,.5,.25),"inch"),
         legend.position = c(.5,1.2),
         legend.text = element_text(size = 8),
         legend.key.height = unit(10, 'points')) +

  theme(axis.text.x = element_text(size = 9)) +
  facet_wrap(~Beach, nrow = 1)

plt
```

<img src="beaches_graphics_files/figure-gfm/trend_plot-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trend_two_beaches.pdf', device = cairo_pdf, 
       width = 5, height = 4)
```