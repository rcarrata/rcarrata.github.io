---
layout: single
title: Custom Backup Satellite 6
date: 2017-08-15
type: post
published: true
status: publish
categories:
- Satellite
- Linux
- Automation
tags: []
author: rcarrata
comments: true
---

Some time ago, a customer ask for a question about Satellite 6.1.x: how about the backups?
 
So, I deep further into the oficial documentation and I found a script that is provided with the satellite installation:
 
```
   # /usr/bin/katello-backup backup_directory
``` 

This script copy and make a tar of every folder described:
 
```
   - /etc/ /var/lib/pulp
   - /var/lib/mongodb
   - /var/lib/pgsql/
```

With this seems to be ok I thought, but when you try to automatize the backups of the satellite it's not enough:
 
```
   - The backups is compressed without timestamp
   - There is not an option for only backup the config files
   - Every backup, saves ALL the rpm, making the files compressed huge (200Gb in some cases)
   - There is not an option for only backup the DB only without pulp data
   - The script not fails when the backup directory is not defined.
   - The script is not valid for backup the capsules
```

So, for do an automatic and unnatended backups I modified the original script and added some new features:
 
```
   - Timestamp for each backup generated
   - Fancy menu for show the usage and the options
   - Option to backup only the DBs
   - Option to backup only the Pulp Data
   - Adapted to backup also the Capsules
   - Backup directory is generated with the timestamp
```

With this [backup satellite script](https://github.com/rcarrata/satellite_backups/blob/master/satellite-backup.sh) and with a simple job in Jenkins that triggers every week at 3pm the backup, the backups are generated and stored.
 
This is tested and is working with Satellite version 6.1.8 and 6.1.9.
 
EDITED: The Satellite in version 6.2 have a new backup script, wrote in ruby that improves some stuff.
 
Hope that helps!
 
*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*
 
Rober


<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="You like this blog? It helped? Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>