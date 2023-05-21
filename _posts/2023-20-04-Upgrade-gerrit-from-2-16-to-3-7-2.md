---
title: Upgrade Gerrit from 2.16 to 3.7.2
date: 2023-04-020 11:03:29 +0100
categories: [Sysadmin]
tags: [sysadmin, linux, gerrit]
math: false
mermaid: false
---

Just though, I would share some of my recent experience of upgrading Gerrit from 2.16.19 to at this time most recent 3.7.2.

1. Stop the Gerrit service and make the backup if you're able to

```service
systemctl stop gerrit
```

Pre-download installation .war files. We will need
the one we are using right now (for me 2.16.19.)
-3.1.2
-3.7.2

```commands
cd /opt
wget https://gerrit-releases.storage.googleapis.com/gerrit-2.16.19.war
wget https://gerrit-releases.storage.googleapis.com/gerrit-3.1.2.war
wget https://gerrit-releases.storage.googleapis.com/gerrit-3.7.2.war
```

> I'm running and compiling everything under the gerrit user
{: .prompt-tip }

```commands
su gerrit
```

Migrate the database
Run ‘migrate-to-note-db’ command to migrate all changes on a separated database to NoteDB

> Note that my installation is stored  in /var/www/gerrit, but yours may vary
{: .prompt-tip }

```commands
java -jar /opt/gerrit-2.16.19.war migrate-to-note-db --threads 4 -d /var/www/gerrit
```

