---
layout: post
title: Estimating Traffic Density for PiinPoint
---

Suppose you want to open a new store for your business, what kind of information might you want to know about the new location?  One key piece, perhaps, is how many cars and people will go by the store?  This is one small part of the *multitude of metrics* the analytics company [PiinPoint](https://www.piinpoint.com/) delivers to its customers.  However, it turns out that this is a tricky business and PiinPoint teamed up with [Insight](http://insightdatascience.com/) (and me) to see if we can dig deeper.

# The Problem

Estimating pedestrian and vehicle traffic is tough - companies rich in the right data seldom give it out.  Moreover, even those who have it often don't have a complete picture.  To tackle this [PiinPoint](https://www.piinpoint.com) has teamed up with [Miovision](https://miovision.com/) a traffic company that does spot measurements of people and autos at various locations around the globe.

The trick is, it's hard for anyone to provide complete spatial-temporal coverage everywhere around the globe.  In Miovision's case this is also true.  For instance, here is where data has been collected (as of this date) for NYC.

![_config.yml]({{ site.baseurl }}/images/nyc_map.png)

Whoa, interesting huh?  From first thought you might think that Miovision would want a uniform coverage of a given area, but that is not what they do!  The aim is to serve their customers who pay to know what's going on at *specific locations*.  

In addition, the above map doesn't paint the full picture.  For a given location, there might be many measurements across each our of the day or there might just be a one for each hour or there might be hours with no measurements at all!  

![_config.yml]({{ site.baseurl }}/images/curves.png)

The above plot shows three random sets of data for six different locations.  The top row shows pedestrian counts and the bottom shows light/medium vehicles (think cars, SUVs, trucks, and vans).  As you can see, even at locations where there are data, it is far from complete.  Some locations do have very nice coverage (like the location on the lower left), while others are much more incomplete.  This is particularly true at the morning hours from midnight to 10 AM.

So what does this mean for our PiinPoint team?  The goal:

> At each location with at least one measurement, estimate the vehicle and pedestrian traffic for hours with no data.

You can imagine that this might be a tricky task given the heterogeneous nature of the data.  Nevertheless our goal is to improve on their models, as measured by the root mean squared error (RMSE) on held-out test data.

# First Cracks

When deciding to take this consulting job on as my Insight project, I was initially intrigued.  What a great opportunity to run some Gaussian Processes with cool kernels (e.g., [these cats](http://arxiv.org/abs/1302.4245)) or perhaps some creative density estimation (e.g., my acquaintance Iain Murray and crew's [RNADE](http://arxiv.org/abs/1306.0186)). Alas, these methods rely on reasonable sampling of the space and after seeing the above views of the data, I switched my focus.

Here at Insight (and as Data Scientists) we often want to *move fast*.  Which often translates to: start simple and build up.  A very simple and easy first step to this problem is K-Nearest Neighbors.  The idea is simple: for a given location, find the closest locations (by distance) to the location of interest and estimate the missing measurement (for a given hour) by taking a weighted mean of the K neighbors (where K is something like 1, 2, or 10).  Weights of the K neighbors are determined by inverse distance, and K is set through cross-validation.

Turns out this didn't work so well.  The Root Mean Squred Error (RMSE), was not too much better than what PiinPoint is currently doing (think averaging by hour over a large spatial area).  As you can image the variation of traffic can be immense from place to place, even moreso for a dense city (like NYC).  Ultimately, this makes the K neighbors somewhat noisy estimators for the traffic counts.

# Building the model

Intuitively, the nearest neighbors to a location must relate in *some* way to the traffic density.  However, that relation might be somewhat complicated.  A second intuition is if you are trying to estimate traffic at a given hour (at a particular location), the hours before and after are STRONGLY correlated with it.  

The idea is as follows. For a given location-hour pair, compute the K nearest neighbors for the hour and the hours immediately before and after.  Take the distances to, and the traffic counts associated with, the neighbors as features X to predict the actual traffic counts Y.  Train a good regression algorithm to improve the RMSE.

Tree-based methods are great for this task - they are fairly robust, and can build complex non-linear relations between the inputs X and the outcomes Y.  Specifically, I chose to try the ensemble methods [Random Forests](https://en.wikipedia.org/wiki/Random_forest) and [Gradient Boosting](https://en.wikipedia.org/wiki/Gradient_boosting).   Here is a look at the performance of these approaches on held-out test data.

![_config.yml]({{ site.baseurl }}/images/rmse_hour.png)

The above shows the RMSE on an hourly basis for four models.  Note the PiinPoint model is significantly worse in terms of error than all other models.

Due to the incredibly sparse nature of the data, I wanted to see if I could augment our features X with publicly available data.  One obvious choice is the US Census.  The intuition is simple - areas with low/higher population ought to correlate with pedestrian and vehicle traffic.  Moreover, the median age of a location might affect traffic patterns (think worklife versus nightlife).  Using the Census Tract data, I constructed an interpolation based method of estimating these quantities for any given location.  Once in place, these quantities were computed and added to our features.

The improvement was significant.  Examine in the above figure the difference between gbm (Gradient Boosting Model) and gbm without census data.  The model with Census data performs significantly better, particularly between the hours of 2 and 6.  These hours are times were we typically have less data, so the augmentation is helping exactly in the way we expected.  Below shows the global performance of our best model gbm versus the PiinPoint approach and gbm without census data.

![_config.yml]({{ site.baseurl }}/images/rse.png)

The above distributions show the Root Squared Error.  Notice our best model (gbm) significantly improves the distribution over the PiinPoint approach, and that adding Census data helps as well.

# Wrap up

I really enjoyed the chance to work with PiinPoint and improve the product they deliver to their customers.  Inded, knowing that these models (done in just a couple weeks) will be integrated into their company is very exciting. The project itself was also rewarding.  Since the data were relatively small and quite sparse, it was fun to explore different models and intuitions of what might work.  My initial thoughts of using more complex and powerful models had to disappear, and instead the dance was one of balance between the nature of the problem and models.  
