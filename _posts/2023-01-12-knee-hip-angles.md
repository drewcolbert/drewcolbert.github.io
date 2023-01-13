---
layout: post
title: "Simple ANOVA on Biomechanical Data"
subtitle: "How Sloped Surfaces Alter Our Walking Mechanics"
background: '/img/posts/knee-hip/knee-hip-bg.jpg'
---




Knee and Hip Kinematics on Sloped Surfaces
================

# Introduction

This is a project that was done by myself and three others during my
senior year at the University of Oregon. This was a biomechanics class
and final project was to run an experiment related to biomechanics using
some techniques and software we had learned about in class. Our group
decided to use a motion tracking software to gather data on clinical
knee and hip angles of subjects while walking on different variations of
sloped surfaces. The slopes we chose were: flat, 4° incline, 8° incline,
4° decline, and 8° decline.

At the time of doing this project, I had not yet been exposed to any
programming languages and I had not yet started my journey down the path
of data science and analytics. I wanted to revisit this project with the
tools and knowledge I have now, and essentially use this data to make
better visualizations and a better report overall.

# Background

What does clinical knee angle mean? What about clinical hip angle?

Clinical knee angle is the angle created by the shin and by extending a
line out from the femur (see picture below for example). Clinical hip
angle is very similar except its the angle created by the femur and by
extending a line through a persons center of mass.

<img align = "middle" src="\img\posts\knee-hip\clinical-knee-angle.jpg" alt="Clinical Knee Angle" width="50%" height = "25%"/>

It has already been shown that lower limb kinematics and walking posture
change when walking on a sloped surface. We already know that during the
initial heel strike, clinical knee angle is greater while walking on an
incline compared to flat ground and decline and during mid strike
clinical knee angle decreases more for an incline than on flat ground.
For hip kinematics, we know there is a Significant increase in hip
extension and flexion when walking uphill compared to flat ground and
there is No change in hip flexion or extension when walking downhill.

# Purpose

These changes in gait are important to everyone as sloped surfaces are a
part of everyday life. These changes in gait are especially important to
those who are injured or are just coming off of an injury, using these
slopes can be advantageous to physical therapist who want to target a
specific movement or muscle group in their patient. It is also important
to track any changes to the center of mass while walking on these slopes
for the elderly population. A drastic change to the center mass would
cause someone to fall, this data can be used as a precursor to show
those who may be at risk of falling when walking on a sloped surface.

# Analysis

To start, we are going to load in a few packages.

``` r
library(readxl)
library(ggplot2)
library(plotly)
library(dplyr)
library(tidyr)
library(BayesFactor)
```

The initial analysis was done in excel, so our data is located in an
excel file. Lets load that in and take a look.

``` r
# read in the excel file
file_path <- "C:\\Users\\drewc\\OneDrive\\Documents\\RStudio\\Portfolio\\Datasets\\Knee Hip Data\\knee_hip_angles.xlsx"
knee_hip_angles <- read_xlsx(path = file_path, sheet = 1)


# the data is currently a tibble, I would prefer a dataframe
knee_hip_angles <- as.data.frame(knee_hip_angles)
knee_hip_angles$Subject <- as.factor(knee_hip_angles$Subject)

str(knee_hip_angles)
```

    ## 'data.frame':    110 obs. of  12 variables:
    ##  $ Subject        : Factor w/ 3 levels "1","2","3": 1 1 1 1 1 1 1 1 1 1 ...
    ##  $ Time           : num  0.0333 0.0666 0.0999 0.1332 0.1665 ...
    ##  $ flat_K         : num  5.1 11.3 13.9 11.6 18.4 ...
    ##  $ four_incline_K : num  9.28 5.29 12.53 18.39 17.21 ...
    ##  $ four_decline_K : num  7.26 10.61 13.82 18.78 17.58 ...
    ##  $ eight_incline_K: num  7.37 10.93 13.94 14.48 15.85 ...
    ##  $ eight_decline_K: num  10.05 7.66 10.9 16.53 16.56 ...
    ##  $ flat_H         : num  86.2 85.9 82.8 78.8 84.7 ...
    ##  $ four_incline_H : num  78.5 68.7 60.8 64.5 64.4 ...
    ##  $ four_decline_H : num  91.7 91.7 86.6 83.6 86 ...
    ##  $ eight_incline_H: num  81.9 79.5 78.7 72.8 66.8 ...
    ##  $ eight_decline_H: num  92 85.5 88.2 85 83.3 ...

Here we see that there are three subjects and a time variable tells us
how far along in the gait cycle the subject is. The other columns are
the measured angles of the knee and hip for each different angle that
was tested.

The data frame looks and works fine, but we can do so much better. For
starters, I don’t want to write out those column names every time I go
to plot the data and making the plots will be difficult since each angle
type is in its own column. To change that this we are going to reshape
our data from the wide format to the long format. This will put all of
the angles into one column which makes plotting them much much easier.

I also would like an easy way to filter out just the knee angles or just
the hip angles, so we can create a new column telling us which part of
the body we are talking about and will help us with filtering. Another
change I want to make is a simple one but one that I do think makes a
difference when trying to understand the graph. I want to change the
time variable to a percentage. Since each observation is one complete
gait cycle, I think it makes more sense to look at it in terms of a
percentage rather than as a unit of time. Once all of that is done we
can create our ggplot object and pass that to ggplotly to make it
animated. This will be done with both the knee and the hip, notice how
easy this is because we were able to create our new “component” column.
I’m going to exclude the code used to create the chart for the hip data,
but it is exactly the same with the filter function changed to hip
instead of knee.

``` r
# this functions allows our data to be saved during the animation
accumulate_by <- function(dat, var) {
  var <- lazyeval::f_eval(var, dat)
  lvls <- plotly:::getLevels(var)
  dats <- lapply(seq_along(lvls), function(x) {
    cbind(dat[var %in% lvls[seq(1, x)], ], frame = lvls[[x]])
  })
  dplyr::bind_rows(dats)
}

# create the ggplot object
p <- knee_hip_angles %>%
  
  # change from a wide df to a long df
  pivot_longer(cols = 3:12,
               names_to = "angle_type",
               values_to = "degrees") %>%
  
  # create the new column using regex to find columns that have _K
  mutate(component = if_else(grepl("_K", unlist(angle_type)), "Knee", "Hip"),
         Time = round(Time/max(Time) * 100, 0)) %>%
  
  # filter by just the knee thanks to our new column
  filter(component == "Knee") %>%
  
  accumulate_by(~Time) %>%
  
  ggplot() +
  geom_line(aes(x = Time, 
                y = degrees, 
                color = Subject, 
                frame = frame),
            show.legend = FALSE) +
  labs(y = "Clinical Knee Angle",
       x = "") +
  facet_grid(.~angle_type, labeller = as_labeller(c("eight_decline_K" = "8° Decline",
                                                    "eight_incline_K" = "8° Incline",
                                                    "flat_K" = "Flat",
                                                    "four_decline_K" = "4° Decline",
                                                    "four_incline_K" = "4° Incline"))) +
  theme(axis.text.x = element_text(angle = 45))


knee_ani <- ggplotly(p, tooltip = c("color", "y")) %>%
  
  layout(showlegend = FALSE) %>%
   
   config(displayModeBar = FALSE) %>%
   
   animation_opts(
     frame = 100,
     transition = 0,
     redraw = FALSE) %>%
   
   animation_slider(
     currentvalue = list(prefix = "% of Gait Cycle: ")
   )
```

<iframe src = '\img\posts\knee-hip\knee-animated.html' height = "500px" width = "100%"></iframe>

The same code as above, just with the hip data instead.

<iframe src = '\img\posts\knee-hip\hip-animated.html' height = "500px" width = "100%"></iframe>

Now that we know what the data looks like and can mess around with the
sliders, its time for some statistics. My first thought here to run an
anova between each of the angles for both the knee and the hip. In order
to do this, there are some assumptions that need to be met:

- Our samples need to be normally distributed or near normally
  distribruted
- Equal variance between each our groups
- Independence between groups and within groups

We already know that the independence assumption is met, so lets check
for normality first, starting with the knee for now.

First I want to assign this updated data frame to a variable. (same code
as above)

``` r
# statistical analysis

knee_data <- knee_hip_angles %>%
  
                pivot_longer(cols = 3:12,
                             names_to = "angle_type",
                             values_to = "degrees") %>%
                
                mutate(component = if_else(grepl("_K", unlist(angle_type)), "Knee", "Hip")) %>%
                
                filter(component == "Knee")
```

``` r
# checking the distributions

ggplot(data = knee_data) +
  geom_density(aes(x = degrees, fill = angle_type, color = angle_type), alpha = 0.2) +
  labs(title = "Destributions of Degrees for each Angle",
       x = "Degrees",
       y = "") +
  scale_fill_discrete(name = "Angle Type",
                       labels = c("8° Decline", "8° Incline", "Flat", "4° Decline", "4° Incline")) +
  guides(color = "none") +
  theme(legend.position = c(0.8, 0.7))


# checking the variance
ggplot(data = knee_data) +
  geom_boxplot(aes(x = angle_type, y = degrees, fill = angle_type, color = angle_type), alpha = 0.2, show.legend = FALSE) +
  labs(title = "Variance For Each Angle",
       x = "",
       y = "") +
  scale_x_discrete(labels = c("8° Decline", "8° Incline", "Flat", "4° Decline", "4° Incline"))
```

<img src="\img\posts\knee-hip\knee-dist.png" width="50%" /><img src="\img\posts\knee-hip\knee-var.png" width="50%" />

Our data is *not* normally distributed. This is okay though as a one way
anova is typically pretty robust when it comes to data not being
normally distributed, but this is something to keep in mind and we will
try something different in a little bit that disregards the normality
clause, so we will press on.

Our variance is looking pretty good from the boxplots, these seem to be
equal and I don’t see the need to run a formal test to double check.

I’m going to skip over showing the code for the hip data because it is
essentially copy and paste but replace “knee” with “hip”. For the hip
data, we do see a normal distribution of the data which is great.
However, the variance is not as easy to determine, so Bartlett’s Test
will be conducted to given a more definite answer on whether the
variation is equal. (More about Bartlett’s Test
[here.](https://www.statology.org/bartletts-test/))

<img src="\img\posts\knee-hip\hip-dist.png" width="50%" /><img src="\img\posts\knee-hip\hip-var.png" width="50%" />

``` r
bartlett.test(degrees ~ angle_type, data = hip_data)
```

    ## 
    ##  Bartlett test of homogeneity of variances
    ## 
    ## data:  degrees by angle_type
    ## Bartlett's K-squared = 37.555, df = 4, p-value = 1.384e-07

As we can see from the Bartlett’s Test, the p-value is very low which
tells me that these samples do *not* have equal variances. Similarly to
the normality of the knee data, one-way anovas are robust in dealing
with data with unequal variances as long as each sample has the same
sample size, which we do in our case.

### The ANOVA

Now that we have done all of our checks, lets move on to the actual
ANOVA.

First, I want to take a look our observed means and see what those look
like when compared between the groups. To do this, I’m going to use the
same boxplots from above, except I’m going to flip the axis and add a
point for the mean for each group.

``` r
means_knee <- aggregate(knee_data, degrees ~ angle_type, FUN = mean)
means_hip <- aggregate(hip_data, degrees ~ angle_type, FUN = mean)

# knee
ggplot() +
  geom_boxplot(data = knee_data, 
               aes(x = angle_type, y = degrees, fill = angle_type, color = angle_type), 
               alpha = 0.2, show.legend = FALSE) +
  
  geom_point(data = means_knee, 
             aes(x = angle_type, y = degrees, color = angle_type), 
             cex = 6, inherit.aes = FALSE, show.legend = FALSE) + 
  
  labs(title = "Knee",
       x = "",
       y = "") +
  
  scale_x_discrete(labels = c("8° Decline", "8° Incline", "Flat", "4° Decline", "4° Incline")) + 
  
  coord_flip()


# hip
ggplot() +
  geom_boxplot(data = hip_data, 
               aes(x = angle_type, y = degrees, fill = angle_type, color = angle_type), 
               alpha = 0.2, show.legend = FALSE) +
  
  geom_point(data = means_hip, 
             aes(x = angle_type, y = degrees, color = angle_type), 
             cex = 6, inherit.aes = FALSE, show.legend = FALSE) +
  
  labs(title = "Hip",
       x = "",
       y = "") +
  
  scale_x_discrete(labels = c("8° Decline", "8° Incline", "Flat", "4° Decline", "4° Incline")) +
  
  coord_flip()
```

<img src="\img\posts\knee-hip\knee-means.png" width="50%" /><img src="\img\posts\knee-hip\hip-means.png" width="50%" />

The points represent the means for each angle. For the knee, we can see
that Flat has a noticeably higher mean than the rest of the angles, but
the rest of the angles appear to be nearly the same. For the hip, we see
that the 4° Decline appears to have a lower mean than the rest, but the
rest all have a similar mean.

Lets see if these differences are significant or not.

``` r
aov_knee <- aov(degrees ~ angle_type, data = knee_data)
summary(aov_knee)
```

    ##              Df Sum Sq Mean Sq F value Pr(>F)
    ## angle_type    4   2061   515.2   1.417  0.227
    ## Residuals   545 198172   363.6

``` r
aov_hip <- aov(degrees ~ angle_type, data = hip_data)
summary(aov_hip)
```

    ##              Df Sum Sq Mean Sq F value Pr(>F)
    ## angle_type    4   1804   451.0   1.686  0.152
    ## Residuals   545 145766   267.5

Both tests returned a low F-value and a p-value greater than 0.05 which
means that we fail to reject the null hypothesis and can conclude that
there is no difference in the means of these different angle for both
the knee and the hip.

# Limitations

There are many limitations to this experiment that may have changed the
outcome of the results. The first one being that this study was only
done on 3 people who were all healthy college students. It would be
beneficial to run this experiment on people of all different ages and
view the differences there. Gather more data is always better as well
because it becomes better to make an inference about the population when
you have more samples.

Another limitation is the surface that was used for walking. Since we
were limited to a classroom, we had to improvise on the walking surface
that was used. We used two narrow, pliable boards that got propped up
when necessary. These boards were not completely stable, which meant
that participants may have altered their normal gait patterns to stable
themselves out. This could have lead to greater flexion recorded than
what was expected.

# Conclusion

Although we found no significant difference in this study, there are a
bunch of real life applications for using these kinematics. This data
could be used to to develop targeted rehabilitation programs to minimize
the aggravations of conditions irritated by walking on an inclined
surface. Along with this, sloped surfaces could be used in preventative
care to help people with injuries avoid further damage. A final use case
would be in the development of lower limb prostheses to allow for
effective sloped motion.

Whether it was found to be significant or note, it is important to note
these changes when treating someone with knee or hamstring pain, or if
you are caring for the elderly. Sloped surfaces are a part of everyday
life and learning how to effectively use them could lead to increased
biomechanical results.

Special thanks to: Megan Quan, Marie Soto, and Jonathan Nahass for
running this experiment with me!
