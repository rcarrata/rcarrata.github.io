---
layout: single
title: Custom Backup Satellite 6
date: 2017-08-15
type: post
published: true
status: publish
categories:
- Satellite
tags: []
author: rcarrata
comments: true
---

Some time ago, a customer asked a question about Satellite 6.1.x: what about the backups?
 
So, I dug further into the official documentation and found a script that is provided with the Satellite installation:
 
```
   # /usr/bin/katello-backup backup_directory
``` 

This script copies and creates a tar of every folder described:
 
```
   - /etc/ /var/lib/pulp
   - /var/lib/mongodb
   - /var/lib/pgsql/
```

This seemed to be fine, I thought, but when you try to automate the backups of the Satellite it's not enough:
 
```
   - The backups are compressed without a timestamp
   - There is no option to back up only the config files
   - Every backup saves ALL the RPMs, making the compressed files huge (200GB in some cases)
   - There is no option to back up only the DB without pulp data
   - The script does not fail when the backup directory is not defined
   - The script is not valid for backing up the capsules
```

So, to create automatic and unattended backups, I modified the original script and added some new features:
 
```
   - Timestamp for each backup generated
   - Fancy menu to show the usage and options
   - Option to back up only the DBs
   - Option to back up only the Pulp Data
   - Adapted to also back up the Capsules
   - Backup directory is generated with the timestamp
```

With this [backup satellite script](https://github.com/rcarrata/satellite_backups/blob/master/satellite-backup.sh) and with a simple job in Jenkins that triggers every week at 3pm the backup, the backups are generated and stored.
 
This is tested and is working with Satellite version 6.1.8 and 6.1.9.
 
EDITED: Satellite version 6.2 has a new backup script, written in Ruby, that improves some of this.
 
Hope that helps!
 
*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*
 
Rober


<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>