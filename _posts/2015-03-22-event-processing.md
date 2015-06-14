---
layout: post
title: Real-time recommenders, part three -  Event tracking (processing events)
tags:
- Recommenders
- Event Tracking
- AWS
---

### Introduction
In order to utilize our event data from Kinesis we have configured two separate feeds: one non-real-time for reporting and archiving purposes and one real time feed for use cases like recommendations and last viewed items. While snowplow has use case specific libraries for both, we built our own utilities for the following reasons:

* **Flexibility:** Building our own library allows easier loading to multiple database engines and/or unique archive flows

* **Speed:** Using a basic loader allows event rotation into the database once a minute without any loss of information.

* **Archive Structure:** The default snowplow libraries do not make any provision for DB loading and archiving to S3; for reliability purposes we wanted both 


### Database loading
In order to process the data for loading into the database we are using a mixture of the default [Kinesis Enricher](https://github.com/snowplow/snowplow/tree/master/3-enrich/scala-kinesis-enrich), sed, a small Python script for event validation, and cronolog to write to a file. The exact flow looks something like:

1. Kinesis enricher stderr and stdout to stdout ```/var/lib/snowplow_enrich/snowplow-kinesis-enrich-0.2.0 --config /var/lib/snowplow_enrich/snowplow.conf 2>&1```

2. sed to remove checkpointing data from the event stream ```sed -e '/^\[/d' | sed -e 's/\[pool.*$//'```

3. A Python script which validates whether or not the event has been processed before and validates the custom JSON objects inside the structured event labels

4. Cronolog which rotates log files every minute ```cronolog /data/logs/snowplow%Y%m%d%H%M.dat```


One issue with using cronolog is that it splits exactly on the minute, resulting in one event per minute being cut in half every minute. Concatenation of the logs before loading (for example, every 15 minutes) reduces the impact to one event per load, but this is still unacceptable - we would like 100% of events being written to the database. To correct, our loader uses a mixture of __head__ and __tail__ to grab the end of the last event for loading from the head of the log file currently being written. Once the log files are correctly merged, loading and backups become trivial:

```
#!/bin/bash

RIGHTNOW=`date +%Y%m%d%H%M`
MINUTEAGO=`date -d '-1 minute' +%Y%m%d%H%M`
YEAR=`date +%Y`
MONTH=`date +%m`
DAY=`date +%d`

find /data/logs -cmin +1 -type f -exec mv "{}" /data/logs/loading/ \;
ls /data/logs/loading/snowplow*.dat | sort | xargs cat> /data/logs/loading/temp.dat
head -n 1 /data/logs/snowplow$MINUTEAGO.dat>>/data/logs/loading/temp.dat
tail -n +2 /data/logs/loading/temp.dat > /data/logs/loading/snowplow_merged_$RIGHTNOW.dat
rm /data/logs/loading/snowplow2*.dat
rm /data/logs/loading/temp.dat

/opt/vertica/bin/vsql -h my.loadbalancer.path -U $1 -w $2 -c "copy stg.snowplow_events from local '/data/logs/loading/snowplow_merged_$RIGHTNOW.dat' with delimiter E'\t'  rejected data '/data/logs/loading/snowplow_rejected_$RIGHTNOW.dat'"

gzip /data/logs/loading/snowplow_merged_$RIGHTNOW.dat

s3cmd put /data/logs/loading/snowplow_merged_$RIGHTNOW.dat.gz s3://bykdwh/snowplow/$YEAR/$MONTH/$DAY/
rm /data/logs/loading/snowplow_merged_$RIGHTNOW.dat.gz
```

Far simpler than the default loaders and extremely performant in our environment. Another advantage to having the log files on a minute (or five minute) basis is that it makes it extremely easy to restore an hour or a day if we need to reload.


### Real time processing
While the process outlined above is fantastic for reporting purposes, a one minute delay on the events is still not fast enough for a "real time" experience for users. Our objective for users is that the data is analyzed and recommendations updated before their next pageview; this gives us five seconds or less to process the event. To achieve this throughput we are using inline processing in a Python script to read specific data from the event, check records against redis, and execute a number of puts  to redis. While this approach is somewhat cumbersome and lower performance than using a language like Scala, it is still possible to achieve throughput of a few thousand events per second per thread - more than sufficient for most businesses.

The functional workflow of real-time processing looks something like:

```python
import sys
import redis
import json
import fileinput

action_value={
	'action1':weight,
	'action2':weight,
	'action3':weight
}


def unlist(input_list):
	return '\t'.join(map(str,input_list))


def process_event(event):
	# Do some event processing to extract the correct user, event type (based on data in the event label), and SKU
	return (user_id, event_type, product_sku)


def main():
	counter = 0
	pipe = rec_set.pipeline()

	for line in sys.stdin:
		try:
			split_tab = []
			split_tab.append(line.split('\t'))
			split_tab = [val for sublist in split_tab for val in sublist]
			user_id, event_type, product_sku = process_event(split_tab)

			if product_sku is not None:
				# Do some lookups for recommendations and brand information, then load those recommendations to redis
			else:
				next
		except:
			next


if __name__ == '__main__':
	main()
```