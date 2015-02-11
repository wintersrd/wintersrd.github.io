---
layout: post
title: Real-time recommenders, part two: Event tracking (front end, collector, and bus)
tags:
- Recommenders
- Event Tracking
- AWS
---

### Introduction
In order to generate recommendations for users, the first task is to generate user data. One of the best options available right now is to use [Snowplow](https://github.com/snowplow/snowplow), an open source event tracker built around the AWS stack. Using Snoplow plus [Google Tag Manager](http://www.google.com/tagmanager/), one can get event tracking running in just a few hours.


### Front End Tracker
Rather rewriting already excellent documentation, simply follow the notes [here](https://github.com/snowplow/snowplow/wiki/Integrating-javascript-tags-with-Google-Tag-Manager). However, a few tips in getting started:

1. **Start small:** Capture Pageviews only and validate against Google Analytics. This will expose potential tracking issues in your front end immediately which might cause tracking issues (we initially were missing almost half of our events due to ajax refreshes).

2. **Be prepared for lots of data:** Without having a baseline, it's easy to underestimate how much data will arrive. When we turned on page pings on a very conservative interval (every two minutes), **event volume  doubled** even though average time on page was well under a minute.

3. **Use the existing event types:** Snowplow is prebuilt to support a number of different events, if at all possible use those rather than "rolling your own" to capture similar data.


### Event Capture and Bus
In order to get real time recommendations, data must be processed in real time; this rules out the "production ready" clojure collector and forces one to use the scala streaming collector. The good news is that this appears to be extremely stable; we have been using it in production for several months with no significant issues. 

Configuration of the collector is [relatively straightforward](https://github.com/snowplow/snowplow/wiki/Setting-up-the-Scala-stream-Collector). Since our event volume can swing significantly during the day, we set up an AWS image which auto-starts the collector and configured an Elastic Beanstalk application for this purpose. The threshold used is CPU >65% for launching an instance and <25% for removing an instance as this was the greatest cause of event processing delays and dropped data. One major advantage of the streaming collector is that machines can be turned on and off with virtually no loss of data; using the clojure collectors, one can auto-scale up but must manually scale down to ensure that web logs rotate correctly.

While it is possible to write the event stream directly to stdout (for easy processing), we write the records to Kinesis as we have two consumer streams on the data set: one for real time recommendations and one for recording the data to the DB and archiving data to S3. One issue with Kinesis is that it does *not* autoscale. A quick estimate of event volume to bus utilization is that one needs one shard for every 10k events per minute, depending on how much additional data is written per event.