---
layout: post
title: Data Vault Automation: Source extraction
tags:
- Data Vault
- Automation
---

Note: These posts on data vault automation are an in-depth discussion of functionality described [in a 10 Mar 2015 presentation on the Bijenkorf's DWH architecture](http://www.slideshare.net/RobWinters1/data-vault-automation-at-the-bijenkorf)


When we began our vault rollout, one of the initial challenges we faced was the disparity of relational systems to work with: MySQL (various versions), Oracle 11g, Vertica, etc. While heterogeneous replication is an ideal solution in this landscape, the majority of servers were partner hosted and so actual code/feature deployment on the machines was not possible. As such, the easiest solution available was to deploy code generators based on metadata derived from our staging environments. This approach offers a number of advantages:

* **Easy integration of new tables:** Rather than building new ETLs or modifying code to add new tables from existing sources, we can integrate the new information automatically by deploying the DDL in the DWH

* **Resiliency against source changes:** In the event that the source system adds additional columns to the source, changes referential integrity, etc, ETLs are not impacted. Only INT->VARCHAR changes is source data models cause issues.

* **Automatic full vs incremental loading:** Using the last timestamp off our ODS view layer allows the scripts to only pull the new records from the sources; in the event that nothing is currently in the ODS, the engine will automatically pull everything.


### Engine workflow
