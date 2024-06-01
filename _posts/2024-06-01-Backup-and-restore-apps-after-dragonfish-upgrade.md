---
title: Backup-and-restore-apps-after-dragonfish-upgrade
date: 2024-06-01 20:00:00 +0100
categories: [Homelab. truenas]
tags: [truenas, cobia, dragonfish, heavyscript]
math: false
mermaid: false
---

# Introduction
I've created this guide after going over all the new documentation from TrueChart, HeavyScript, and many Discord threads. I've qouted a LOT. I don't mean to steal all of their work or anything like that, but I wanted to create a more simple guide with all of the options you currently have and debug them as I've tried them on my own system.

> Migration to a new platform is not yet complete, and frankly, you should just wait until 1.7.2024 to make any plans for upgrading to Dragonfish, in my opinion. After that it should be more clear what the plan is and how to do it.
{: .prompt-note }

## Options for now

#### A. heavyscript DragonFish Backup/Restore Beta Release (branch DF-backup-restore)
[Github-link](https://github.com/Heavybullets8/heavy_script/tree/DF-backup-restore)

#### B. truechart half-baked backup option to S3 storage (RESTORE DOES NOT WORK YET)


> I highly suggest going for the A option for now, as truechart restore is still not working (as of the publication of this post). And the heavyscript single app restore just works beautifully.
{: .prompt-warning }


# A. Heavyscript DragonFish Backup/Restore Beta Release
I'm going to quote Heavy himself from the discord and combine it with the new heavyscript github readme and all of the discussion info I've read to provide some of my notes.

[Link to the discord announcement and discussion] (https://discord.com/channels/830763548678291466/1088745486267187251/1242964345550540961)

* switch to beta branch (select DF-backup-restore)
```bash
sudo bash heavy_script.sh git --branch
```

#### * new backup/restore options
* * -c [number] or --create [number]: Create a backup with the specified retention number.
* * -r or --restore: Restore a backup.
* * -d or --delete: Delete a backup.
* * -l or --list: List all backups.
* * -i [backup_name] [app_name] or --import [backup_name] [app_name]: Import a specific backup for an application.

```bash
#example
heavyscript backup --create 10
```

### Comparison Table from heavyscript git

| Feature                            | Full Backup            | Export/Import       |
|------------------------------------|------------------------|---------------------|
| Persistent Volume Claims (PVCs)    | ✔️                    | ❌                  |
| CNPG Databases                     | ✔️                    | ❌                  |
| Official App Support               | ✔️                    | ❌                 |
| TrueCharts Apps                    | ✔️                    | ✔️                 |
| Supported by TrueCharts             | ❌                    | ✔️                 |
| Snapshotting PVCs                  | ✔️                    | ❌                  |


### How to Choose
__Full Backup and Restore:__ Choose this method if you need a comprehensive backup of the entire app system, including CNPG databases and PVCs. This method ensures that all data is captured, but it is more resource-intensive and may take longer to perform.

> A good replacement for IX's API Backup/Restore if you use Truecharts apps.
{: .prompt-note }


__Export and Import:__ This method is recommended if you only need to back up and restore application values. It is faster and less resource-intensive but it does not capture PVCs or databases. This method is officially supported by TrueCharts.


> By default, both export and full backup are enabled in the configuration file. You can disable them by editing the ~/heavy_script/config.ini file.
{: .prompt-note }


## Setup backup/restore
### Create full backup interactivly  
heavyscript -> Backup options -> Create backup -> Specify the retention


### Cron job for automatic backup/update apps
Because the backup option doesn't support the update flag, we need to combine the new option with the old one, which only backups the ix-application dataset.

```bash
heavyscript backup --create 14 && heavyscript update --concurrent 10 --prune --rollback --sync --self-update --include-major --major
```

### Restore single app
heavyscript -> Backup options -> Restore single backup -> leavy empty to select interactivly -> select app



## Encountered issues
### ValueError: Dataset Speed/heavyscript_backups does not exist
*  create heavyscript_backups ZFS dataset in the same pool as your APPS dataset


# B. Truechart backup with VolSync
[Truechart guide](https://truecharts.org/scale/guides/backup-restore/)
[Discord discusion](https://discord.com/channels/830763548678291466/1234481318499323956)

Truechart has decided to go with [VolSync](https://github.com/backube/volsync), which uses rsync/rclone to asynchronously replicate Kubernetes persistent volumes between clusters.


> VolSync depends on Prometheus-Operator, so ensure you have installed Prometheus-Operator prior to installing VolSync.
{: .prompt-note }



## Correctly Setup OpenEBS
Edit OpenEBS app
In the App Configuration make sure ZFS-LocalPV volumeSnapshotClass
* is enabled
* isDefault

And __deletionPolicy__ is __Delete__.

![OpenEBS](/assets/img/posts/2024-05-31-Backup-charts-and-pvc-after-dragonfish-upgrade.md/open_ebs.png)


## Install VolSync chart
Apps -> Discover Apps -> VolSync

Make sure it's the one from system train as there is also one from incubator train.


## Setup S3 storage
Both CloudFlare and BackBlaze provide a free plan with 10GB of storage included, but I'll go with CloudFlare.

* Register/Login to [cloudflare](https://www.cloudflare.com/)
* Click on R2 storage
* Agree with subsciption
* Click on manage api tokens
* Create API token and name it to your liking
Permissions: Admin Read & Write
             TTL: Forever


It should look something like this
```
800XXXXXXXXXXXXXXXXXXXXXb54edc82b3
963f10dfb4fXXXXXXXXXXXXXXXXXXXXXXa6445e704b53b943ee9
https://6745XXXXXXXXXXXXXXXXX281822ea56.eu.r2.cloudflarestorage.com
https://67XXXXXXXXXXXXXXXXXXXXXXXa56.r2.cloudflarestorage.com
```

> Dont forget to copy Access Key ID & Secret Access Key & URL
{: .prompt-warning }


## Setup credentials for backup (for each app)

Warning
Do not add the credentials inside the VolSync chart. This won’t work, as they need to be added to each chart individually.

Open up desired app (for example prowlarr) and enter your S3 credentials under the credentials section.

You have to do this for each and every app you want to have backup/restore functionality on.

![credentials](/assets/img/posts/2024-05-31-Backup-charts-and-pvc-after-dragonfish-upgrade.md/credentials.png)



## Setup PVC backup (for each app)

PVC data can also be backed up to S3 storage with VolSync support.

"For each individual chart, the VolSync Destination (Restore) option must be set on the creation of the app by doing the following::"
So you need to recreate all of your existing charts according to this
* Add VolSync to each persistence object you want synced, as below:
* Add the name you gave to the S3 credentials earlier, under the credentials (IN THE SAME CHART YOU ARE CURRENTLY SETTING UP volSync)
* Enable the VolSync Source (backup) and/or VolSync Destination Restore) options as desired
* Confirm the data is being sent to your S3 host after ~5 minutes




![busy](/assets/img/posts/2024-05-31-Backup-charts-and-pvc-after-dragonfish-upgrade.md/volsync_pvc.png)


> You do not have to manually create the bucket beforehand, although this is recommended to ensure the bucket’s name is available beforehand  (it should not be a problem with Cloudflare).
{: .prompt-note }


As mentioned in the truchart post, the restore functionality isn’t yet functional on TrueNAS SCALE due to an [upstream bug](https://github.com/openebs/zfs-localpv/issues/536) with OpenEBS.


```bash
[EFAULT] Failed to update App: WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /etc/rancher/k3s/k3s.yaml Error: UPGRADE FAILED: cannot patch "jackett-config" with kind PersistentVolumeClaim: PersistentVolumeClaim "jackett-config" is invalid: spec: Forbidden: spec is immutable after creation except resources.requests for bound claims   core.PersistentVolumeClaimSpec{    ... // 4 identical fields    StorageClassName: &"openebs-zfs-main",    VolumeMode: &"Filesystem", -  DataSource: nil, +  DataSource: &core.TypedLocalObjectReference{ +  APIGroup: &"volsync.backube", +  Kind: "ReplicationDestination", +  Name: "jackett-config-config-dest", +  }, -  DataSourceRef: nil, +  DataSourceRef: &core.TypedObjectReference{ +  APIGroup: &"volsync.backube", +  Kind: "ReplicationDestination", +  Name: "jackett-config-config-dest", +  },   }
```

To get around this, uncheck __restore option__ and wait until it's fixed in the future.


## CNPG Database Backups

CNPG-backed PostgreSQL databases have their own S3 backup system. Truechart has integrated it in such a way that they can safely share a bucket with the above PVC backups.

For each app:
* Add CNPG backups to each database you want backed up like shown below.

* Add the name you gave to the S3 credentials earlier, under the credentials section.

* Confirm the data is being sent to your S3 host after ~5 minutes

> The official guide also mentions that you should "Set the “mode” to restore, this should prevent the app starting with an empty database upon restore". DO NOT DO THIS. First of all it's a typo and it should be __recovery__ and it's only meant for the recovery process, not for the backup.
{: .prompt-warning }



# Truechart restore with VolSync
When you’ve no exported app configuration, you can remake the app while also restoring your PVC and CNPG backups using the steps as follows:

* Ensure the app name matches the name of the app previously backed up

* Enter the same S3 credentials from earlier, under the credentials section.

* Preferably, ensure all other configuration options are set precisely the same as the last time you used the app, to ensure compatibility.


> PVC data restoration will happen automatically before the app starts.
{: .prompt-note }


## CNPG DB Restore
Before CNPG will correctly restore the database, the following modifications need to be done after recreating or importing the app configuration:

* Ensure you’ve setup CNPG backups as well as restored them as it were previously (in the backup step).

* Ensure the “mode” is set to recovery (while creating the app)

* Set “revision” on your restore to match the previous revision setting on your backup settings (That means you should only type in the credentials if you have followed this tutorial)

* Increase the revision on your backup setting by 1 (or set to 1 if previously empty) (default is empty)