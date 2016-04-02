---
layout: post
title: Real-time recommenders, part one - Overview of Technology
tags:
- Recommender
---
### Introduction
This is the first in a series of posts on how to implement near-real-time tracking and personalization for your users based on off the shelf and open source technologies. While many organizations perceive "real time" as challenging, the solution outlined here has been tested to scale easily to ten million pageviews per day; significantly higher volumes than most organizations encounter.

### Fundamental components
In order to create recommendation for your users, a few fundamental components must be present in your technical environment:

* **Event Tracker:** A mechanism to capture user activity in near real time and make it available for scoring/analysis.

* **Data Platform:** The environment where your models will be calculated and history stored. Typically column store databases or Hadoop are used.

* **Data Presentation Platform:** Where the user recommendations will be stored and presented from. High throughput of reads/writes is critical here.

* **Front End Implementation:** How your users will consume the recommendations from your models. 

That's it, regardless of whether you are personalizing for ten users or ten million. From start to finish, an entire recommender architecture can be developed and deplyoed in about a week. In the 
next post I'll discuss implementation of Snowplow, an open source event tracking stack.