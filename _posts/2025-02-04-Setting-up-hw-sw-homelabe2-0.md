---
title: Setting up Hardware && Software [Homelab 2.0]
date: 2025-02-04 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0]
math: false
mermaid: false
series: homelab-2.0
---

# Introduction
After we've [successfully chosen](https://blog.thetechcorner.sk/posts/Hw-Sw-choices-alternatives/) what components we want to use and what software to run, we can start building it.

## In this post I'll show you how to
### Hardware
* mode Fractal node 304 front panel to fit 200mm noctua fan
* put all components together
* connect new power button to the motherboard

## Software
* flash bios on Cwwk q670 plus [optional] 
* do basic BIOS configuration on Cwwk q670 plus
* install/configure proxmox hypervisor
* import/create ZFS pools (SSD pool for apps, HDD pools for NAS storage)



# How to
### Mode Fractal node 304 front panel
The original Fractal node 304 comes, with the fully covered front panel. I've found this to be extremely bad for the airflow in the case, so I decided it has to go.

Fortunately, there is a relatively easy solution (which also looks amazing!) and we can put big 200mm noctua fan in.

I'm talking about [this Node 304 front panel mod](https://www.printables.com/model/137181-200mm-fan-front-for-fractal-node-304/files) by pixelwave, which also made its appearance in [Wolfgang NAS building video](https://www.youtube.com/watch?v=wwsgCkbogMM) 

1. cut the hole   
   
![cutting_hole](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/cut_hole.jpg)

2. print the parts 

I went with the black/brown color combination in the classic Noctua color scheme.   
If you look closely, I've put one more [PVC mash filter](https://www.aliexpress.com/item/1005006245902609.html?spm=a2g0o.order_list.order_list_main.29.21ef1802qgGLi6) behind brown 3d printed one, to keep the dust out.


![printed_parts](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/printed_parts.jpg)

You can find some amazing color combination ideas [in the comments](https://www.printables.com/model/137181-200mm-fan-front-for-fractal-node-304/comments) as well!

 
3. put the Noctua fan in   
  
Be mindful of the orientations. Follow the arrows on the fan. So the air is getting pulled from the front panel and pushed to the case to left where the exhaust is
   
![airflow](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/airflow.jpg)

![noctua_fan](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/noctua_fan.jpg)


4. put the power button in the case

You can buy the same button I've used from [amazon](https://www.amazon.de/dp/B09FSLP2QQ)

Simply push the power button in the the front panel hole and connect the cables, as you can see in the picture.

![button_wiring1](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/button_wiring1.jpg)


### Put all components together
#### Put the disks inside the "white sockets"
![together0](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together0.jpg)

For SSD I've used this [SSD caddy](https://www.printables.com/model/342894-node-304-ssd-caddy) 3d printed mod from @DrJekyll_242009

![ssd_caddy](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/ssd_caddy.png)

#### Insert CPU, cooler, RAM, GPU to the motherboard.

![together1](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together1.jpg)

#### Put PSU in the the case and attach all the power cables to motherboard and disks.

The motherboard requires both 24pins+8pins ATX 12V
![together2](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together2.jpg)


![together3](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together3.jpg)


![together4](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together4.jpg)



#### Connect power button to motherboard
![button_how_to2](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/button_how_to_2.png)

![button_how_to](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/button_how_to.png)


### Basic BIOS configuration
[PDF manual](/assets/text/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/StoneStormQ670H670ProductManual-2.pdf) to the older version of the same motherboard. 

For better power efficiency we can enable C states. 

> In order to save energy when the CPU is idle, the CPU can be commanded to enter a low-power mode. Each CPU has several power modes and they are collectively called “C-states” or “C-modes.”.
The lower the state the deeper the sleep is and the less power the CPU takes.
{: .prompt-info }


Advanced > Power & Performance > CPU > C states = Enabled
Advanced > Power & Performance > CPU > Package C state Limit = C10
Auto start on boot = Enabled

I didn't find any other BIOS settings to make notable difference.


##### To enable features like ASPM, you need to used edited BIOS (or edit BIOS yourself) -> follow bellow





### Flash bios on Cwwk q670 plus [optional] 
Cwwk has disabled multiple settings in the BIOS (like ASPM, SATA DevSlp ...) because of problems related to that setting features (instead of fixing it..)

Fortunately, the community got together once again and created new edited BIOS version with re-enabled features. 

> This may potentially brick or make device buggy, so do this on your own risk.
{: .prompt-warning }

1. Download default (non-edit version) [11/18 firmware] from [cwwk website](https://pan.x86pi.cn/BIOS%E6%9B%B4%E6%96%B0/3.NAS%E5%AD%98%E5%82%A8%E7%B1%BB%E4%BA%A7%E5%93%81%E7%B3%BB%E5%88%97BIOS/9.%E7%AC%AC12-13-14%E4%BB%A3-AlderLake-RaptorLake-Desktop-%E5%8F%8C%E7%BD%91H670-Q670-NAS/AlderLake-RaptorLake-%E5%8F%8C%E7%BD%91_12-13-14%E4%BB%A3_Q670-NAS_20241108%E6%9B%B4%E6%96%B0)


2. Flash the default version to the USB (with something like rufus)

3. Download edited BIOS version created by [Yonji1](https://www.reddit.com/user/Yonji1/) from [google drive](https://drive.google.com/file/d/1Y9ADVtSWtUgXvOApZCGxHf1yPa4KmdIu/view)

3. replace the files on the flashed USB

4. boot the system with USB inserted to flash


Now you should be able to use ASPM. Just be careful, because users has reported errors like 
```bash
errors like 'pcieport 0000:00:1d.0: AER: Corrected error received' #on the nvme slot 
```

``` bash
- It looks like ASPM substates are not really working, L1 seems to be fine but L1.1 and L1.2 probably doesn't work (are grayed out in BIOS). I even forced it with nvram default settings to L1.1 & L1.2 but no difference really.

- NVME in front slot looks to be working with L1.

- PCIE root #3 (and #2 too probably) are the ones for network card. If you enable ASPM there most likely you won't be able to get your network interface up. 
```

```bash
Looks like Intel I226-V is limiting ASPM. Trying to activate ASPM causes a system freeze. Not sure what is the cause of this
```

> If anything happens, just fall back and flash to default BIOS version
{: .prompt-info }

#### Edit BIOS yourself

With this free opensourced [UEFI-edit tool](https://github.com/BoringBoredom/UEFI-Editor/tree/master) you can even edit BIOS yourself. Here is a little [guide](https://winraid.level1techs.com/t/guide-enabling-hidden-bios-settings-on-gigabyte-z690-mainboards/94039) that can help you get started.



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


# Conclusion
With the hardware mods and software configurations now in place, your Homelab 2.0 setup is ready to go. In the following posts, I'll show you what LXC I'm running and how to install/configure them!



## Check out other posts in the series
{% include post-series.html %}