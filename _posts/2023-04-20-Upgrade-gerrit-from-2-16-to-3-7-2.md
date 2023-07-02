---
title: Upgrade Gerrit from 2.16 to 3.7.2
date: 2023-04-20 11:03:29 +0100
categories: [Sysadmin]
tags: [sysadmin, linux, gerrit]
math: false
mermaid: false
---

I decided I would share some of my recent experience with upgrading Gerrit from 2.16.19 to at this time most recent 3.7.2.

## 1. Stop the Gerrit service and make the backup if you're able to

```bash
systemctl stop gerrit
```

Pre-download installation .war files. We will need
the one we are using right now (for me 2.16.19.)
* 3.1.2
* 3.7.2

```bash
cd /opt
wget https://gerrit-releases.storage.googleapis.com/gerrit-2.16.19.war
wget https://gerrit-releases.storage.googleapis.com/gerrit-3.1.2.war
wget https://gerrit-releases.storage.googleapis.com/gerrit-3.7.2.war
```

> I'm running and compiling everything under the gerrit user
{: .prompt-tip }

```bash
su gerrit
```

## Migrate the database
Run ‘migrate-to-note-db’ command to migrate all changes from old database to NoteDB

> Note that my installation is stored  in /var/www/gerrit, but yours may vary
{: .prompt-tip }


```bash
java -jar /opt/gerrit-2.16.19.war migrate-to-note-db --threads 4 -d /var/www/gerrit
```

When you run ‘migrate-to-note-db‘ command, you must have to set ‘–threads’ as small as possible. If you don’t set it, a migration process creates too many threads to make connections to a database and you could see some error messages
</br>
</br>

## Disable the review of the database
```bash
vim /var/www/gerrit/etc/notedb.config
disableReviewDb = true
```

## Upgrade the java
```bash
yum install java-11-openjdk.x86_64 -y
update-alternatives --config java
#select the new downloaded 11 openjdk

#change the path to the new java in the gerrit config
vim /var/www/gerrit/etc/gerrit.config
[container]
user = gerrit
javaHome = /usr/lib/jvm/jre-11-openjdk
```

## Initialize and reindex to a new 3.1.2 version

> I’ve faced some issues when I tried to jump so many versions at 1 time, so we are doing 2 jumps
{: .prompt-tip }

```bash
java -jar /opt/gerrit-3.1.2.war init -d /var/www/gerrit
java -jar /opt/gerrit-3.1.2.war reindex -d /var/www/gerrit --index changes
```

### Start the gerrit and check the status/errors
```bash
systemctl start gerrit
systemctl status gerrit
```

if everything is ok, and you get no errors, stop the Gerrit again and continue.


```bash
systemctl stop gerrit
```

I got an error complaining about the “rename-project” plugin, I decided to delete it  this time and install it at the end with the new version

```bash
rm -rf /var/www/gerrit/plugins/rename-project.jar
```

## Upgrade to 3.7.2

```bash
#download the missing plugin
cd /var/www/gerrit/plugins/
wget https://gerrit-ci.gerritforge.com/job/plugin-rename-project-bazel-master-stable-3.5/lastStableBuild/artifact/bazel-bin/plugins/rename-project/rename-project.jar

#init and reindex to a new version
java -jar /opt/gerrit-3.7.2.war init -d /var/www/gerrit
java -jar /opt/gerrit-3.7.2.war reindex -d /var/www/gerrit --index changes

systemctl start gerrit
systemctl status gerrit
```