---
title: Cobia to Dragonfish Upgrade
date: 2024-05-30 20:00:00 +0100
categories: [Homelab, truenas]
tags: [truenas, cobia, dragonfish]
math: false
mermaid: false
---


# Introduction
This post is about my experience with the Truenas upgrade from Cobia to Dragonfish and the problems I've encountered.

Here is the official [guide](https://truecharts.org/scale/migrations/cobia-dragonfish/)

# Changes
* iX-Systems no longer maintains or supports any form of PVC-based storage for apps
* * heavyscript backup and restore will no longer back up PVC storage at all
* * no PVC storage rollback
* * no PVC pool migration

# Prereq

## 1. Backup and update all apps with heavyscript
```bash
heavy_script update --backup 14 --concurrent 10 --prune --rollback --sync --self-update --include-major --major
```

##  2. Change apps to the correct train
With the "new train: premium,"  you need to migrate the following charts:
traefik, blocky, authelia, grafana, nextcloud, vaultwarden, prometheus, clusterissuer, or any other app that you use custom-app chart
*  Add premium train to truechart catalog (Apps -> Discover Apps -> Manage catalogs -> Click the truechart catalog -> Add preferred train "Premium")
*  Run official migraton script https://truecharts.org/news/#note-automated-migration-script-for-train-name-changes
```bash
curl -sSL https://raw.githubusercontent.com/xstar97/scale-scripts/main/scripts/patchTCTrains.sh | bash --
```

If that won't work (happend to me on custom-app) try this

```bash
k3s kubectl get ns
k3s kubectl patch ns ix-<appname> -p '{"metadata":{"labels":{"catalog_train":"premium"}}}'
```

## 3. Upgrade to dragonfish via GUI
In the truenas GUI go to System settings -> Update -> Select Dragonfish Update

> Wait at least 30 minutes after updating and the SCALE WebUI becomes accessible
{: .prompt-warning }

> Logout/login again to see apps running or you will keep seeing STATUS Deploying
{: .prompt-info }

## 4. Delete old kubernetes objects
```bash
rm /mnt/<REPLACE>/ix-applications/k3s/server/manifests/zfs-operator.yaml && sudo k3s kubectl delete -f https://truecharts.org/openebsrem.yaml && sudo k3s kubectl delete storageClass openebs-zfspv-default

rm /mnt/pool-fast/ix-applications/k3s/server/manifests/zfs-operator.yaml && sudo k3s kubectl delete -f https://truecharts.org/openebsrem.yaml && sudo k3s kubectl delete storageClass openebs-zfspv-default
```

## 5. Create Dataset for PVCs and Install OpenEBS
Create a dedicated, empty dataset on the apps pool you created above, outside of ix-applications should look like this

![Datasets](/assets/img/posts/2024-05-30-Cobia-to-dragonfish-upgrade.md/datasets.png)

> When setting the pool/dataset as above, do not set the path to the existing ix-applications dataset
{: .prompt-warning }

>The dataset you create for OpenEBS usage MUST reside on the same storage pool as the existing ix-applications dataset. Refer to the example apps dataset images at the end of the guide.
{: .prompt-info }

## 6. Install OpenEBS
Truechart new OpenEBS storage solution
[Official guide](https://truecharts.org/scale/#openebs-setup)

* First we need to add system pref train in truechart catalog (Apps -> Discover Apps -> Manage catalogs -> Click the truechart catalog -> Add preferred train "System")

 * Install OpenEBS chart
  Configure: * reclaim policy: Retain
             * pool/data:  poolname/dataset-name  (without mnt/) in my case pool-fast/pvcApps



## 7. Migrate Apps to New Storage using [TT-Migration Script](https://github.com/Heavybullets8/TT-Migration)

The script will do following tasks (which you can do manually if you want) https://github.com/Heavybullets8/TT-Migration
* find the apps pool
* find the app (from the name you provided)
* check if app is on valid train, if it's on system train (in which case it won't be migrated),
* check if app is on correct pool (datase ix-applications has to be on same pool as OpenEBS chart)
* check if app is using CNPG database
* create migration and app datasets
* rename app's PVC to new path  <pool-name>/migration/<appname>/<appname>-backup
* stops and delete old app
* delete old app dataset

#### As a bonus, script also migrates the db, you can either
* * let script dump db and migrate it (Default)
* * provide your own db dump, or heavyscript path to db dump

```bash
git clone https://github.com/Heavybullets8/TT-Migration.git && cd TT-Migration && sudo bash migrate.sh
```
#### Type in the app name and press Enter

#### To run it once again without cloning the repository
```bash
sudo bash migrate.sh
```

If one of the following tasks fail, the script will provide you with steps to fix the error.





# Workarounds for errors:
## Dataset busy
![busy](/assets/img/posts/2024-05-30-Cobia-to-dragonfish-upgrade.md/dataset_busy.png)

## This chart doesn't appear anywhere in the catalog
##  The chart: <chartname> is not in the current train: enterprise.
Follow step 2. in this post
Example

In my case I'm running sonnar in the custom app
```bash
root@truenas[~/TT-Migration]# k3s kubectl patch ns ix-sonarr -p '{"metadata":{"labels":{"catalog_train":"premium"}}}'
namespace/ix-sonarr patched
```
Run the bash TT-mig script again

## Error: Failed to create the application after 3 retries.
In this case traefik didn't completely migrated and we need to fix a few things
If we check error log
```
    "repr": "CallError('Unable to locate \"26.7.3\" catalog item version.')",
```

And as we can see there is no such version in their github repository
https://github.com/truecharts/catalog/tree/main/premium/traefik
![versions](/assets/img/posts/2024-05-30-Cobia-to-dragonfish-upgrade.md/github_traefik.png)


We can either manually rewrite the backupfile stored in
```bash
/mnt/<pool-name>/migration/traefik/
```

or use --latest-version flag in combination  with --skip which will list all apps that didn't completely migrate
```bash
sudo bash migrate.sh --latest-version --skip
```
