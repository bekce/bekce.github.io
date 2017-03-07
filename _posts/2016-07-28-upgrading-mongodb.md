---
layout:     post
title:      Upgrading MongoDB from 2.x to 3.x
date:       2016-07-28 12:26:00
categories: sysadmin
---

___Update___: This guide actually works with migrating from *ANY* version to *ANY* version (including latest v3.4) on a MongoDB supported Ubuntu version.

If you have installed MongoDB with the default version from package maintainer version (for example Ubuntu 14.04), then you are using v2.4 which is pretty much an archaic version and MongoDB has improved greatly particularly with WiredTiger or RocksDB storage engines since then. If you read the upgrade instructions on the official documentation, you would have seen that you need to follow this version chain 2.4 -> 2.6 -> 3.0 -> 3.2 to upgrade. Wait, what? In practice, it just does not work out. In this post, I will put my tested way of upgrading directly from 2.4 to 3.2 (or latest) AND switch to WiredTiger storage engine also.

### Workflow

1. Dump whole database with `mongodump` as bson
2. Backup log, data and config files (for being extra careful)
3. Purge (remove) old version packages and everything from the system
4. Install the newest version via package manager
5. Restore data using `mongorestore` tool

### Exact List of Commands

{% gist 4070a67dc23d003653d3af1833e1bb1e upgrade_mongodb.sh %}
