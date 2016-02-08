---
layout: post
title: Estimating Traffic Density for PiinPoint
---

Suppose you want to open a new store for your business, what kind of information might you want to know about the new location?  One key piece, perhaps, is how many cars and people will go by the store.  This is one small part of the *multitude of metrics* the analytics company [PiinPoint](https://www.piinpoint.com/) delivers to its customers.  However, it turns out that providing traffic counts is a tricky business, so PiinPoint teamed up with [Insight](http://insightdatascience.com/) (and me) to see if we can dig deeper.

# The Problem

Estimating pedestrian and vehicle traffic is tough - companies rich in the right data seldom give it out.  Moreover, even those who have it often don't have a complete picture.  To tackle this problem [PiinPoint](https://www.piinpoint.com) has teamed up with [Miovision](https://miovision.com/) a traffic data company that does spot measurements of people and automobiles at various locations around the globe.

The trick is, it's hard for anyone to provide complete spatial-temporal coverage everywhere around the globe.  In Miovision's case this is also true.  For instance, here is where data has been collected (as of this date) for NYC.

![_config.yml]({{ site.baseurl }}/images/nyc_map.png)

Whoa, interesting huh?  From first thought you might think that Miovision would want a uniform coverage of a given area, but that is not what they do!  The aim is to serve their customers who pay to know what's going on at *specific locations*.  

In addition, the above map doesn't paint the full picture.  For a given location, there might be many measurements across each our of the day or there might just be a one for each hour or there might be hours with no measurements at all!  

![_config.yml]({{ site.baseurl }}/images/curves.png)

The above plot shows Miovision data for six random locations.  The top row shows pedestrian counts and the bottom shows light/medium vehicles (think cars, SUVs, trucks, and vans).  As you can see, even at locations where there are data, it is far from complete.  Some locations do have very nice coverage (like the location on the lower left), while others are much more incomplete.  This is particularly true at the morning hours from midnight to 10 AM.

So what does this mean for the PiinPoint team?  They need a model which provides traffic estimates at times without direct measurements.  

> *GOAL:* At each location, estimate the vehicle and pedestrian traffic for hours with no data.

You can imagine that this might be a tricky task given the heterogeneous nature of the data.  Nevertheless our goal is to improve on their models, as measured by the root mean squared error (RMSE) on held-out test data.

# First Attempts

When deciding to take this consulting job on as my Insight project, I was initially intrigued.  What a great opportunity to run some Gaussian Processes with cool kernels (e.g., [these cats](http://arxiv.org/abs/1302.4245)) or perhaps some creative density estimation (e.g., my acquaintance Iain Murray and crew's [RNADE](http://arxiv.org/abs/1306.0186)). Alas, these methods rely on reasonable sampling of the space and after seeing the above views of the data, I switched my focus.

Here at Insight (and as Data Scientists) we often want to *move fast*.  Which often translates to: start simple and build up.  A very simple and easy first step to this problem is K-Nearest Neighbors.  The idea is simple: for a given location, find the closest locations (by distance) to the location of interest and estimate the missing measurement (for a given hour) by taking a weighted mean of the K neighbors (where K is something like 1, 2, or 10).  Weights of the K neighbors are determined by inverse distance, and K is set through cross-validation.

Turns out this didn't work so well.  The Root Mean Squared Error (RMSE), was not too much better than what PiinPoint is currently doing (think averaging by hour over a large spatial area).  As you can image the variation of traffic can be immense from place to place, even more so for a dense city (like NYC).  Ultimately, this makes the K neighbors somewhat noisy estimators for the traffic counts.

# Building an initial model

Intuitively, the nearest neighbors to a location must relate in *some* way to the traffic density.  However, that relation might be somewhat complicated.  A second intuition is if you are trying to estimate traffic at a given hour (at a particular location), the hours before and after ought to be well correlated with it.  

The idea is as follows. For a given location-hour pair, compute the K nearest neighbors for the hour and the hours immediately before and after.  Take the traffic counts and distances associated with the neighbors as features <code>X</code> to predict the actual traffic counts <code>Y</code>.  Train a good regression algorithm to improve the RMSE.

Tree-based methods are great for this task - they are fairly robust, and can build complex non-linear relations between the inputs <code>X</code> and the outcomes <code>Y</code>.  As a first stab, I chose to try the ensemble method [Random Forests](https://en.wikipedia.org/wiki/Random_forest).  Taking this approach proved to be a great improvement over the baseline model from PiinPoint (see *Model Exploration* below).  Using a Random Forest on the k-neighbor pairs improved the RMSE to about 1.4 times better than PiinPoint's current approach.

# Further Improvements: Augmenting the data

Due to the incredibly sparse nature of the data, I wanted to see if I could augment our features <code>X</code> with publicly available data.  The motivation for this is simple - the Miovision data is fairly sparse in both space (location) and time, so we want to provide our predictive model additional information to supplement the places were sparsity is the worst.

One obvious relevant data source is the US Census.  The intuition is simple - areas with low/higher population ought to correlate with pedestrian and vehicle traffic.  Moreover, the median age of a location might affect traffic patterns (think work-life versus night-life).  Using the Census Tract data, I constructed an interpolation based method of estimating these quantities for any given location.  Once in place, these quantities were computed and added to our features.

Next, I had the intuition that the type of street that a location is on might be  important.  Intuitively, highways are likely to have more vehicle traffic than a 'court' or a 'lane'.  The opposite may be true perhaps for pedestrians. So I queried the [Google Geocode API](https://developers.google.com/maps/documentation/geocoding/intro) and placed the road types into the following bins:

<code><p style="color:#6E6E6E"><sub><sup>{highway/route, street/road/drive, lane/place/court/way/circle, ave/blvd, bridge/tunnel, path/walk/bridge}</sup></sub></p></code>

Finally, I wanted to try to inform the model about the nearby density of interesting places that might draw traffic.  For this information I turned to [Factual.com](https://factual.com/), who have data on the location and types of places near a given latitude and longitude.  The categories for places are broad at the high level, but sub-categories can be quite specific.  Due to API limits and time constraints I was limited to putting places into two bins.  The first one is more business/services-like places:

<code><p style="color:#6E6E6E"><sub><sup>{Automotive, Community/Government, Healthcare, Business/Services, Travel}</sup></sub></p></code>

The second is a bin for more recreation-like places:

<code><p style="color:#6E6E6E"><sub><sup>{Retail, Landmarks, Restaurants/Bars, Sports/Rec, Landmarks,
		  Social, Transportation}</sup></sub></p></code>

For both of these bins, I got counts from factual.com within 50, 150, and 300 meters from the location of interest.

# Model Exploration

Now armed with a more rich set of data, it was time to explore the space of models that would suit PiinPoint's problem.  The goal for this stage is to produce a model that performs well, but also one that is simple to implement for PiinPoint.  

The first set of models I considered were based on the [scikit-learn](http://scikit-learn.org/stable/) library, since they are stable, work well, and have a consistent API.  For basic comparision purposes I first considered a K nearest neighbors model only.  This performed fairly poorly (see table below), yielding just a 5% improvement over PiinPoint's current model.

Next, I considered the ensemble methods [Random Forests](https://en.wikipedia.org/wiki/Random_forest) and [Gradient Boosting](https://en.wikipedia.org/wiki/Random_forest).  For both of these, I varied the features and parameters put into the model and used cross-validation to determine the best model.  The table below highlights some of the models explored.

![_config.yml]({{ site.baseurl }}/images/model_table.png)
<sub><sup><center><p style="color:#6E6E6E">C = Census data, S = Street type, and F = factual.com place data</p></center></sup></sub>

In addition to the scikit-learn model library, I wanted to explore two other types of models which often perform well for estimation tasks like the one here.
Specifically I tried Bayesian Additive Regressive Trees (sampled with [Particle Gibbs](http://www.gatsby.ucl.ac.uk/~balaji/pgbart_aistats15.pdf)) and Neural Networks (built with [Keras](http://keras.io/)).  While these models seemed promising, they failed to deliver the best RMSE on test data.  I believe with more time the model parameters and/or network architecture could be tuned to deliver great performance, but given the time allotted for this project and the goal of delivering something simple, I abandoned further exploration.

# Results

Examining the above table it is clear that the gradient boosting model performed best on held out test data.  For pedestrians the model gave a RMSE of 334 (when combined with US Census data) and for vehicles the model yields a RMSE of 537.  More importantly these values are **~50%** the RMSE that PiinPoint's current model gives!

> *DELIVERED:* A model that is TWICE as accurate than before!

This is a great result, but what does it mean (practically) for PiinPoint?  Have a look at the below:   
![_config.yml]({{ site.baseurl }}/images/rse.png)

These histograms show the residuals (errors) from model predictions and the test data.  On the left are the residuals for the best vehicle model and PiinPoint's vehicle model.  Strikingly, the PiinPoint distribution has a lot of points where the errors are of order 1000 counts or greater.  Intuitively, this scale of error (+/- 1000) seems like it would be a big deal to there customers since this scale is 1) a large amount of traffic and 2) more than the typical variation seen at locations.  Now, PiinPoint will typically give there customers estimates that are off by a few hundred (or better) counts!

The panel on the right of the figure shows the best vehicle model (which uses data from the census, street types, and factual) versus a model with the census information dropped.  As you can see, the effect of adding in Census (or other data) is more subtle and certainly not as dramatic as the left panel.  Given this, and the correspondingly smaller effect on RMSE, I recommended to PiinPoint to carefully consider whether it was worthwhile to add in these additional data sources - it might be better practically to keep things simpler and exclude some external data (at the cost of slightly worse error).

# Wrap up

I really enjoyed the chance to work with PiinPoint and improve the product they deliver to their customers.  Indeed, knowing that these models (done in just a couple weeks) will be integrated into their company is very exciting. The project itself was also rewarding.  Since the data were relatively small and quite sparse, it was fun to explore different models and intuitions of what might work.  My initial thoughts of using more complex and powerful models had to disappear, and instead the dance was one of balance between the nature of the problem and models.  
