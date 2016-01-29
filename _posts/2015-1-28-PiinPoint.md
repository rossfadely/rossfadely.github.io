---
layout: post
title: Estimating Traffic Density for PiinPoint
---


Suppose you want to open a new store for your business, what kind of information might you want to know about the new location?  One key piece, perhaps, is how many cars and people will go by the store?  This is one small part of the *multitude of metrics* the analytics company [PiinPoint](https://www.piinpoint.com/) delivers to its customers.  However, it turns out that this is a tricky business and PiinPoint teamed up with [Insight](http://insightdatascience.com/) (and me) to see if we can dig deeper.

# The Problem

Estimating pedestrian and vehicle traffic is tough - companies rich in the right data seldom give it out.  Moreover, even those who have it often don't have a complete picture.  To tackle this [PiinPoint](https://www.piinpoint.com) has teamed up with [Miovision](https://miovision.com/) a traffic company that does spot measurements of people and autos at various locations around the globe.

The trick is, it's hard for anyone to provide complete spatial-temporal coverage everywhere around the globe.  In Miovision's case this is also true.  For instance, here is where data has been collected (as of this date) for NYC.

![_config.yml]({{ site.baseurl }}/images/censored.png)
