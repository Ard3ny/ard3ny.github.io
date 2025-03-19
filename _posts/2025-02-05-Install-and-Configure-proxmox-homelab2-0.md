---
title: Install && Configure Proxmox [Homelab 2.0]
date: 2025-02-05 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0, proxmox, LXC]
math: false
mermaid: false
series: homelab-2.0
---

# Introduction
After we've successfully installed hardware in place and edited BIOS, we can proceed with the proxmox installation

## In this post I'll show you how to
## Software
* flash bios on Cwwk q670 plus [optional] 
* do basic BIOS configuration on Cwwk q670 plus
* install/configure proxmox hypervisor
* import/create ZFS pools (SSD pool for apps, HDD pools for NAS storage)



# How to
### Install/Configure Proxmox (PVE)
There is A LOT of guides telling you how to install/configure proxmox so I'll be brief

1. download the newest [PVE ISO](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)

2. flash the iso to USB (with something like rufus)

3. boot system with USB inserted

4. install the system on desired IP, drive (mine being 512GB SSD)

5. login at https://<IP>:8006
#### Post-install configuration
We are going to run few post install proxmox scripts created by community (previously by tteck)

1. [Proxmox VE Post Install](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install)

2. [Proxmox VE Processor Microcode](https://community-scripts.github.io/ProxmoxVE/scripts?id=microcode)


### ZFS introduction
The best article on ZFS I've come across by far [arstechnica](https://arstechnica.com/information-technology/2020/05/zfs-101-understanding-zfs-storage-and-performance/)

It explains the whole zfs structure (pool, vdev, dataset, blocks...), how copy-on-write works and many more, so everything I would write here would just be copy or inferior explanation.


> Just remember, that you issue "zfs" commands against datasets and "zpool" commands against pools and vdevs.
{: .prompt-info }



#### Import/Create ZFS pools 
We will create/import 2 pools
* pool-fast (SSD storage for all LXCs')
* pool-01 (main HDD storage that will contain stuff like photos, movies..)

##### create ZFS pool
Go to node -> Disks -> ZFS -> Create: ZFS
Choose name, raid level.

We can create pool either over CLI or GUI
##### GUI
Datacenter -> Storage -> Add -> ZFS

##### CLI
```bash
#create mirror pool
zpool create pool-01 c1t0d0 c1t1d0

#create raid pool
zpool create pool-01 raidz c1t0d0 c2t0d0 c3t0d0
```

##### Import [optional if not creating new]
1. in proxmox node shell (CLI) type
```bash
#to list all possible options (if there are multiple)
zpool import

#specify which pool to import
zpool import <pool-name>

#If the pool was not cleanly exported, ZFS requires the -f flag to prevent users from accidentally importing a pool that is still in use on another system
zpool import -f <pool-name>
```


#### Create zfs dataset 
ZFS datasets are a powerful and flexible organizational tool that let you easily and quickly structure your data, monitor size over time, and take backups.


We can either create dataset over CLI or GUI
##### GUI
Datacenter -> Storage -> Add -> ZFS

##### CLI
```bash
zfs create pool-01/test
zfs list pool-01/test
```

> In later post I'll explain to you possibility of installing software like cockpit to manage your ZFS share and permissions.
{: .prompt-info }

##### ZFS scrubs 
Should be enabled on proxmox by default. You can check by running
```bash
crontab -e 
```

#### ZFS Arc limit

By default, ZFS uses 50% of the host memory for the Adaptive Replacement Cache (ARC). (Since >8.2 PVE installation should be defaultly set to 10% of installed RAM [max 16 GiB]).


ARC is very useful for IO performance. You may ask, why change it? 


ZFS ARC is an in-memory caching system that speeds up disk access by keeping frequently and recently used data readily available. It dynamically balances between recency (data accessed recently) and frequency (data accessed often) by using two lists—one for most recently used data and another for most frequently used data. This adaptive approach ensures that the cache efficiently adjusts to the workload, improving overall system performance.


Unfortunately, it doesn't help that much in my homelab NAS scenario, where I either

* watch a movie (won't watch it again for a while, so caching is pretty much useless.)

* tinker with stuff like Kubernetes and new open-source tools

* run LXC for stuff like pi-hole, arr stack etc... 

* or just run stuff on SSDs and caching doesn't help that much 

  
  
  
It may be useful for scenarios like those where you frequently access nextcloud/immrich/bookstack instances, where the caching provides a great buffer for slow HDD speeds. 



> Rule of thumb according to some people on reddit and forums is: use 2GB + 1GB for every 1TB of effective storage you will be using. For example if effective storage 8TB, then you should use 10GB of ARC.
{: .prompt-info }


 I leave it at approximately 15%, which 10GB for me.


##### Check the current ARC limit
Should be 0 by default

```bash
cat /sys/module/zfs/parameters/zfs_arc_max
```

##### How much RAM is currently ARC taking 
```bash
awk '/^size/ {printf "ARC size: %.2f GiB\n", $3/(1024*1024*1024)}' /proc/spl/kstat/zfs/arcstats
```


##### Some "math" first
I've 64Gb RAM.  
15% of that is approximately 10 Gb.


##### Apply for current boot
Change the value "10" if you want to use more/less RAM.  
For 10 GB run

```bash
echo "$[10  10241024*1024]" >/sys/module/zfs/parameters/zfs_arc_max
```




##### Apply to make it persistent.
```bash
echo "$[10  10241024*1024]" > /etc/modprobe.d/zfs.conf
```




##### How much RAM is currently ARC taking now?
You should see new value now.

```bash
cat /sys/module/zfs/parameters/zfs_arc_max
```

> 
If your root file system is ZFS, you must update your initramfs every time this value changes with "update-initramfs -u -k all" and reboot.
{: .prompt-warning }


Sources [1](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#sysadmin_zfs_limit_memory_usage) [2](https://www.dlford.io/memory-tuning-proxmox-zfs/#3-arc-tuning-etcmodprobedzfsconf)




#### Swap on ZFS
Swap is a type of virtual memory that acts as an overflow area when your system’s physical RAM is full. When the operating system runs out of available memory, it temporarily moves less-used data from RAM to the swap space (which can be a dedicated partition or a swap file on your disk), freeing up RAM for active processes. This helps prevent out-of-memory errors.


This indeed prevents out-of-memory errors but significantly reduces performance. (because SSD is much slower than RAM)


By default, Proxmox is using vm.swappiness=60, which is a lot in my opinion. And if you are going to use ZFS like me, it actually causes more problems.

The swappiness value is scaled in a way that confuses people.

The number doesn't represent any simple value, like when vm.swappiness=60, this means start swapping when 60%RAM is used. 60 roughly means don't swap until we reach about 80% memory usage, leaving 20% for cache and such. 50 means wait until we run out of memory. Lower values mean the same, but with increasing aggressiveness in trying to free memory by other means. You can check out swapping behavior in this [article](https://lwn.net/Articles/83588/)


According to [proxmox documentation](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#zfs_swap), the value to improve performance when sufficient memory exists in a system is "vm.swappiness = 10". I would argue that swap on ZFS is very bad idea and it's not supported.


TLDR
* If you are running PVE root OS system on ZFS don't use SWAP (running swap file on ZFS dataset is the worst scenario)
* If you wanna be on the safe side of things and you expected higer RAM spikes which cloud lead too OOM use "vm.swappiness = 10".   
* If you wanna "more aggresive" approach go with "vm.swappiness = 1" (Which uses minimum amount of swapping without disabling it entirely.)

I will go with "vm.swappiness = 1".

##### Find our where your swap file exists
Mine for example is stored under /dev/dm-o on non-zfs filesytem. So in reality I should be OK with vm.swappiness = 10, but I dont deem it necessary.

```bash
swapon -s
Filename                                Type            Size            Used            Priority
/dev/dm-0                               partition       8388604         546048          -2
```
##### Change swapiness value
Check the current swap value

```bash
cat /proc/sys/vm/swappiness
```


Change to new value

```bash
vim /etc/sysctl.conf
```

```bash
#add the line
vm.swappiness = 1
```


Apply changes
```bash
sysctl -p
```




##### [Optional] Clear swap space
If your system is currently swapping (it really shouldn't on a fresh install), we can clear swap so the new upper limit of "swappiness = 10" is applied (or just reboot)  

```bash
swapoff -a && swapon -a
```


### KSM Tuning
Kernel Samepage Merging (KSM) is a Linux kernel feature that performs memory deduplication by scanning for and merging identical memory pages shared between different processes or virtual machines. Instead of storing multiple copies of the same data, KSM identifies duplicate pages and consolidates them into a single copy, using copy-on-write semantics to ensure that any changes by one process don’t affect others. This can significantly reduce memory usage, especially in virtualization environments where many instances might run similar operating systems or applications.    


In PVE KSM can optimize memory usage in virtualization environments, as multiple VMs running similar operating systems or workloads could potentially share a lot of common memory pages.   


The default value for KSM_THRES_COEF is 20, which means KSM will trigger if less than 20% of the total system RAM is free.

If I've 64GB, the KSM kicks in at 12.8Gb (give or take), which means when the used RAM is approximately =>50GB   


However, while KSM can reduce memory usage, it also comes with some security risks, as it can expose VMs to side-channel attacks. Research has shown that it is possible to infer information about a running VM via a second VM on the same host, by exploiting certain characteristics of KSM. Thus, if you are using Proxmox VE to provide hosting services, you should consider disabling KSM, in order to provide your users with additional security. Furthermore, you should check your country’s regulations, as disabling KSM may be a legal requirement.   


[sourced from pve wiki](https://pve.proxmox.com/wiki/Kernel_Samepage_Merging_(KSM))


But in the homelab scenario, especially if you won't expose your PVE to internet, kSM can be useful, so the choise is yours.  


##### Check current page sharing value  
Should be 0, if you are not hitting 80% of RAM utilization.   

```bash
cat /sys/kernel/mm/ksm/pages_sharing
```




#### [Optional 1] Disable KSM
```bash
#disable KSM
systemctl disable --now ksmtuned

#unmerge pages
echo 2 > /sys/kernel/mm/ksm/run
```







#####  [Optional 2] Change the KSM value
```bash
vim /etc/ksmtuned.conf
```

Uncomment and change the line "#KSM_THRES_COEF" to xy. For example 35%  

```bash
KSM_THRES_COEF=35
```





### Add email notifications to proxmox
Using email notifications in Proxmox is a great way to monitor your cluster, disks, backups and anything that PVE deems necessary to inform you.  

You will get notify in case of:  

* disk failure  
* backup failure  
* power outage (if using UPS) - for this checkout my upcoming UPS proxmox article   



All of these are very important pieces of information you need to have, if they occur. For example, if you are running RAIDZ1, and you lose disk and you don't replace is as soon as possible, your next disk failure will probably cause you to lose all data!!   



Check out the technotim [article](https://technotim.live/posts/proxmox-alerts/) and [video](https://www.youtube.com/watch?v=85ME8i4Ry6A) on email notifications in proxmox topic. I'll pretty much just summarize it.




#### Create a dedicated gmail account  
I highly recommend not using your main, as it provides a huge security risk.   




#### Create app password for gmail account
[Here is official google documentation on how to do that](https://support.google.com/accounts/answer/185833?hl=en)   

##### TLDR   
1. Sign in to your account   
2. [Go to Create and manage your app passwords link](https://myaccount.google.com/apppasswords)   
3. Fill out app name   


You should receive a string looking somewhat like this, but different! This is your app password, which we will use later   

```
nllot yoee tdfd pavue
```




#### Open PVE shell
Install dependencies.

```bash
apt update
apt install -y libsasl2-modules mailutils
```




##### Add your smtp credentials and hash them. Change email/password values to yours!
```bash
echo "smtp.gmail.com proxmoxlogger3000@gmail.com:nllotyoeetdfdpavue" > /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd
postmap hash:/etc/postfix/sasl_passwd
```




##### Configure postfix
```bash
vim /etc/postfix/main.cf
```

```bash
relayhost = smtp.gmail.com:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options =
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_tls_CAfile = /etc/ssl/certs/Entrust_Root_Certification_Authority.pem
smtp_tls_session_cache_database = btree:/var/lib/postfix/smtp_tls_session_cache
smtp_tls_session_cache_timeout = 3600s
```


Reload postfix configuration. 
```bash
postfix reload
```




##### Send test email
Change xxxx to the value of the email in which you want to RECEIVE the email.  

```bash
echo "This is a test message sent from postfix on my Proxmox Server" | mail -s "Test Email from Proxmox" xxxx@gmail.com
```


##### Change backup notifications to failure only
In PVE GUI go to   
Datacenter -> Backup -> Edit backup job   


And set the "Send email to" = "On failure only"



##### [Optional] Test the disk failure
I would also recommend pulling out the SATA cable from one off your disks to see if you receive an email. This is kind of dangerous, so, do it only if you know what you are doing. 
But personally, I wouldn't trust this if I didn't try.


# Conclusion
With the hardware mods and software configurations now in place, your Homelab 2.0 setup is ready to go. In the following posts, I'll show you what LXCs I'm running and how to install/configure them!



## Check out other posts in the series
{% include post-series.html %}