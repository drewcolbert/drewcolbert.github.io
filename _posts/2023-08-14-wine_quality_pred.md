---
layout: post
title: "Wine Quality Prediction (R)"
subtitle: ""
background: '/img/posts/wine-quality/wine-bg.jpg'
---



Wine Quality Prediction
================
Drew Colbert
2023-08-14

# Project Summary

This project set out to determine if we could predict the ratings of a
wine based on the chemical make up of the wine and certain intrinsic
properties of the wine. To do this we found a data set containing ~6500
ratings of both red and white wine and 11 intrinsic properties of the
wine. Each wine received a rating of 1-10, 10 being excellent and 1
being bad. In order to try and predict the rating, a randomForest model
was trained and tested on the data with the goal of obtaining the
highest test accuracy possible. Different transformations and inputs
were tried with our randomForest model to try and boost the accuracy. I
found that I was able to predict with 83% accuracy the rating of a wine.
The biggest factor that determined whether or not a wine was rated
highly was the level of alcohol in the wine. As the alcohol amount went
up, so did the rating on average.

This finding was tested to see if it was just by chance or not. An
independent, two-tailed t-test was run on the alcohol levels of wines
rating ‘High’ (7-10) and ‘Low’ (1-6). I found that the average alcohol
level in the ‘High’ rated wines was significantly higher than that of
the ‘Low’ rated wines. With a difference in the average alcohol level
being between 0.114 and 0.110 mL which was found to be significant with
a p-value of 2.2 x 10^-16, suggesting that alcohol does play a big
influence in the rating that a wine receives. A one way ANOVA was run to
find if the mean alcohol level differed between individual rankings. It
was found that ratings 9 and 8, 9 and 7, 4 and 3, 5 and 3, 6 and 3 were
the only ones that did not show a significant difference in alcohol
level. This coincides with our t-test as these ratings fall within the
same ‘High’ or ‘Low’ rating and would not necessarily be expected to be
very different.

# Project Details

### Data Preview

    ## 'data.frame':    6497 obs. of  13 variables:
    ##  $ fixed.acidity       : num  7.4 7.8 7.8 11.2 7.4 7.4 7.9 7.3 7.8 7.5 ...
    ##  $ volatile.acidity    : num  0.7 0.88 0.76 0.28 0.7 0.66 0.6 0.65 0.58 0.5 ...
    ##  $ citric.acid         : num  0 0 0.04 0.56 0 0 0.06 0 0.02 0.36 ...
    ##  $ residual.sugar      : num  1.9 2.6 2.3 1.9 1.9 1.8 1.6 1.2 2 6.1 ...
    ##  $ chlorides           : num  0.076 0.098 0.092 0.075 0.076 0.075 0.069 0.065 0.073 0.071 ...
    ##  $ free.sulfur.dioxide : num  11 25 15 17 11 13 15 15 9 17 ...
    ##  $ total.sulfur.dioxide: num  34 67 54 60 34 40 59 21 18 102 ...
    ##  $ density             : num  0.998 0.997 0.997 0.998 0.998 ...
    ##  $ pH                  : num  3.51 3.2 3.26 3.16 3.51 3.51 3.3 3.39 3.36 3.35 ...
    ##  $ sulphates           : num  0.56 0.68 0.65 0.58 0.56 0.56 0.46 0.47 0.57 0.8 ...
    ##  $ alcohol             : num  9.4 9.8 9.8 9.8 9.4 9.4 9.4 10 9.5 10.5 ...
    ##  $ quality             : Factor w/ 7 levels "3","4","5","6",..: 3 3 3 4 3 3 3 5 5 3 ...
    ##  $ wine_type           : Factor w/ 2 levels "red","white": 1 1 1 1 1 1 1 1 1 1 ...
    ## NULL

### Distributions

Most of the values here are left skewed with very long tails to the
right. This tells me that there could be some outliers that may
influence the ratings. Transformations can be made on the data that can
reduce the amount of left skew that is seen in this data.

<img src="\img\posts\wine-quality\distributions.png" style="display: block; margin: auto;" />

### Correlations

All boxes highlighted in grey are relationships with a correlation value
of 0.40 or greater, indicating the correlation is noteworthy. Alcohol
and density are highly correlated with a value of -0.69. However, the
most important one that relates to our goal is alchol having a
correlation of 0.44 to the quality. This is the only factor that is this
significantly correlated to the quality, followed by the volatile
acidity at -0.27. This will be something to keep an eye on when we are
making our predictions.

<img src="\img\posts\wine-quality\correlations.png" style="display: block; margin: auto;" />

## Predictions

Our predictor variable is quite imbalanced. Most of the ratings are 5 or
6, this will be something that will need to be addressed for
predictions. Even the groups are highly imbalanced.

<img src="\img\posts\wine-quality\group_counts.png" style="display: block; margin: auto;" />

From our correlation matrix, we saw that alcohol was the most correlated
to the quality. Here we see a very interesting pattern. It is clear that
the the average amount of alcohol does go up as the quality goes up.
This is good, this means alcohol will probably be the main factor in
determining the quality of a wine.

<img src="\img\posts\wine-quality\stripchart1.png" style="display: block; margin: auto;" />

<img src="\img\posts\wine-quality\stripchart2.png" style="display: block; margin: auto;" />

### Modeling

I will be creating 10 different randomForest models, with different
transformations and input variables and comparing the accuracy of each.
The following models will be made:

- **Default**: No transformations or changes to the input variables
- **Log Transformed**: Our inputs were skewed left, by taking the log of
  each input, we make them more normally distributed
- **Normalized**: Z-score normalization is performed on the inputs
  before training
- **More Trees**: Increased the number of trees in the model to 1500
  instead of 500
- **Simple**: Use less input variables, only use the top 5 inputs that
  effect the outcome the most
- **Balanced**: Use over and under sampling techniques to balance out
  the number of observations for each group to 500
- **Combination**: The data will be normalized and only the top 5 input
  variables will be used
- **XGBoost**: Multi softmax XGBoost will be used to predict the quality
  level
- **Grouped**: Instead of predicting the exact rating, predict the group
  (High, Medium, Low)
- **Balanced Grouped**: Use over and under sampling on the groups to
  make 1000 observations in each group

<img src="\img\posts\wine-quality\models.png" style="display: block; margin: auto;" />

### Results

The Balanced, Balanced Grouped, and XGBoost all performed the best with
test accuracy levels in the mid to low 80s. The grouped randomForest
would not be the best choice to use because we lose the precision on our
predicitons and we only gain about 4% of accuracy, which to me is not
worth it. So the XGBoost model or the balanced model would be the best
and would yield the best results. The balanced model runs quicker than
the XGBoost does, so that would be the best one to use due to the better
performance.

------------------------------------------------------------------------

The goal of the company is to make wines that are rated at least a 7 or
above. But these ratings can be so subjective and it could be that these
wines were rated 7 or above by chance, and the amount of alcohol had
nothing to do with it. To test this we are going to split the data into
a ‘High’ rated group (7-10) and a ‘Low’ rated group (1-6). A t-test will
be used to compare the means of these groups. Here we can see the
distibutions for each group, right away we can see there could be a
difference, but lets look further to confirm.

<img src="\img\posts\wine-quality\mean_dists.png" style="display: block; margin: auto;" />

The results of our t-test show that we can reject the null hypothesis,
and we can say that the difference in the average amount of alcohol
between the ‘High’ rated group and the ‘Low’ rated group is *not* 0.
There is a significant difference in the means of each group, and
running this test 100 times guarantees the actual difference in the mean
falls within 0.1008 and 0.1141 95 times.

<img src="\img\posts\wine-quality\mean_diffs.png" style="display: block; margin: auto;" />

To further explore and become more precise, an ANOVA was run on each
individual rating to see if differences were found within and between
each rating. The ANOVA resulted in an F-statistic of 320.4 and a
coinciding p-value of 2 \* 10^-16. Since the ANOVA was significant, a
TukeyHSD test was implemented and plotted below to find out exactly
which groups have a significant difference and which ones do not. The
bars colored red means that the null hypothesis was *not* rejected and
there is not a difference in the mean alcohol levels.

What we see is that only those wins that are closely rated (9-8, 9-7,
4-3) are the ones are not significant.

<img src="\img\posts\wine-quality\anova.png" style="display: block; margin: auto;" />
