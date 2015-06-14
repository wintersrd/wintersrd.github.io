---
layout: post
title: Automating Tableau Backups via AWS
tags:
- Tableau
- AWS
---
### Introduction
Tableau is far and away my favorite Business Intelligence tool for its ease of development and performance on small (<20M row) extracts, but I have always found the backup approach disappointing. Each backup can account for a substantial amount of disk space, there is no default backup rotation, and Windows offers insufficient tooling to effectively handle these tasks. Enter [boto](https://boto.readthedocs.org/en/latest/), the Python library for AWS. Using boto plus S3 and tabadmin, we were able to build a backup solution for Tableau which is:

* Infinitely space scalable

* Fully automated and can email on failure

* Handles daily/weekly/monthly backup rotation automatically. Currently we keep daily for two weeks, weekly for six months, and monthly for an indefinite period.

* Low cost. As of February 2015 the storage cost of S3 is approximately $0.03 per month, so keeping backups only runs a few tens of dollars per year.


### Components
There are several critical functions to generating and rotating the backups effectively:

**1. Options manager:** To read configuration details like AWS keys, bucket names, file naming conventions, etc 

**2. Backup extractor:** To pull the backup, in this case just a system call to tabadmin 

**3. Uploader:** Used to place the file in S3 and validate its placement. Given the file sizes, this must be a multipart upload

**4. Backup Rotator:** Used to eliminate backups which no longer fit our retention policies


### Options Management
Options were handled using the argparse and option parser libraries to store configuration details on our Tableau server; our configuration options were:

```
[aws]
key: 
secret: 
bucket: 
days_to_keep_daily: 14
days_to_keep_weekly: 180


[Tableau] 
tempdir: C:/BACKUPS/
tabadmin_path: C:/Program Files/Tableau/Tableau Server/8.3/bin/tabadmin.exe
filename_base: TABLEAU_BACKUP
```

This affords the flexibility to easily change rotation schedule in the future.


### Backup Extraction
Handled using a simple system call to the tabadmin command based on the parameters set in the config file:

```python
def generate_extract(configs):
	call(configs.get('Tableau','tabadmin_path') + " backup -d " + configs.get('Tableau','tempdir') + configs.get('Tableau','filename_base'))
```

### Uploading and validating
To upload the backup and validate the object, we built a number of small functions to upload the file and confirm its placement. Our files are laoded in a naming convention S3://{bucket}/{tableau_path}/{year}/{month}/{filename_root}-YYYY-MM-DD.tsbak which makes it easy to recover the appropriate backup. Specific functions used include:

1. A multipart file uploader which is "borrowed" wholesale from [boto's S3 examples](http://boto.readthedocs.org/en/latest/s3_tut.html#storing-large-data)

2. A small function to validate the upload was successful using bucket.list()


### File rotation
Since we are backing up daily but have a tiered backup strategy, the most trivial way to do so is to generate a list of valid file names and remove any which do not conform to the list. Specifically, we will need:

1. All file names which fit in the daily retention window (14 days ago through today)

2. All file names which match a Sunday (chosen snapshot day) and are between six months ago and today 

3. All file names which have "01" as the date, regardless of year/month 

As a day can fit 0-3 of these conditions, the easiest solution was to generate three distinct lists, merge them, and deduplicate them using the set function. We can then pull a list of all keys from S3 for our Tableau backups, filtering them, and then deleting the undesired files. One caveat associated with S3 keys from boto: the file name is treated is the entire path after the bucket, so it is necessary to parse out the relevant information for finding the correct file. To do so, the function below is used:

```python
def remove_old(connection, configs):
	keys = connection.list('tableau_backups')
	valid_keys = keys_to_keep(connection, configs)
	keylist = []
	for key in keys:
		keyname = key.key.rsplit('/',1)[1]
		if keyname not in valid_keys:
			keylist.append(key.key)
	for j in keylist:
		connection.delete_key(j)
```

### Conclusion
In addition to the functionality outlined above, it's also helpful to add in email notification in the event of failure and some additional logging. That said, this utility massively reduces data loss risks with Tableau and introduces a useful, low cost mechanism to maintain any easily restorable history of the server.