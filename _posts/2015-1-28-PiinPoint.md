---
layout: post
title: Estimating Traffic Density for PiinPoint
---


Suppose you want to open a new store for your business, what kind of information might you want to know about the new location?  One key piece, perhaps, is how many cars and people will go by the store?  This is one small part of the *multitude of metrics* the analytics company [PiinPoint](https://www.piinpoint.com/) delivers to its customers.  However, it turns out that this is a tricky business and PiinPoint teamed up with [Insight](http://insightdatascience.com/) (and me) to see if we can dig deeper.

# The Problem

Estimating pedestrian and vehicle traffic is tough - companies rich in the right data seldom give it out.  Moreover, even those who have it often don't have a complete picture.  To tackle this [PiinPoint](https://www.piinpoint.com) has teamed up with [Miovision](https://miovision.com/) a traffic company that does spot measurements of people and autos at various locations around the globe.

The trick is, it's hard for anyone to provide complete spatial-temporal coverage everywhere around the globe.  In Miovision's case this is also true.  For instance, here is where data has been collected (as of this date) for NYC.

![_config.yml]({{ site.baseurl }}/images/censored.png)

Whoa, interesting huh?  From first thought you might think that Miovision would want a uniform coverage of a given area, but that is not what they do!  The aim is to serve their customers who pay to know what's going on at *specific locations*.  

In addition, the above map doesn't paint the full picture.  For a given location, there might be many measurements across each our of the day or there might just be a one for each hour or there might be hours with no measurements at all!  

![_config.yml]({{ site.baseurl }}/images/curves.png)

The above plot shows three random sets of data for six different locations.  The top row shows pedestrian counts and the bottom shows light/medium vehicles (think cars, SUVs, trucks, and vans).  The light grey points show a hour's collection of counts, and the red points show the median for the bins in places with more than one measurement.

 
