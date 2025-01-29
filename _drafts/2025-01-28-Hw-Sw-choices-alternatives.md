---
title: Hardware/software choices/alternatives [Homelab 2.0] 
date: 2025-01-28 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0]
math: false
mermaid: false
series: homelab-2.0
---


# Introduction
Follow up to [My-homelab introduction](https://blog.thetechcorner.sk/My-Homelab/)

I've touched on the choices topic already a little, but let's dig a little deeper for the enthusiasts like myself. I will list out final choices and the reason for them, as well as some alternatives.   
I'll also add some useful links and guides I've come across my research.



# Hardware choices
## Motherboard
Motherboard is probably the most important piece in this NAS/homelab build. It dictates pretty much every other part in the system   
* what CPU we can use and thos how powerfull and power efficient the system can be?
* how much RAM (and what kind) we can use?
* how much storage we can have?w
* will we need another network card or we can use PCIE for GPU?
* how big the case has to be?
* etc..

I've decided to go with [CWWK N13 q670 Plus](https://cwwk.net/collections/nas/products/q670-8bay-nas-mini-itx-motherboard-upgraded-version-lga1700-supports-intell12-14-gen-processors-ddr5-dual-4k-displays-5x-usb3-2-8-sata3-0-ports-i226lm-2-5g-with-vpro-q670-2xsff-8643?variant=46801665622248) for several reasons
* mini ITX (fit's in the small case)
* 8x SATA ports
* 3x Nvme
* PCIe5.0 x16 for GPU (for hardware encoding)
* 2x 2.5G LAN
* can be used in combination with [Noctua NH L9i 17xx cooler](https://noctua.at/en/nh-l9i-17xx) 
* relatively cheap

But there are also some disadvantages:
* chinese motherboard, so support is not great
* lack of documentation
* I wouldn't expect many updates
* NO ASPM support (possible to overcome with bios flash)

Even with all these disadvantages, there is just no better value at this moment(which is not scalped). It would be much better to go with some new gen AMD CPUs mobos, but there are just non with so many SATA ports (and I'm not a fan of HBA/SATA extension cards)

##### Sidenote
Q670 Plus is newer version of Q670 V1.
The main difference being the change from 8xSATA3.0 to 2x8643 (4x SATA3.0 /slot)

![q670_compare](/assets/img/posts/2025-01-28-Hw-Sw-choices-alternatives.md/q670_compare.png)


#### Useful links
[CWWK N13 q670 Plus alixpress](https://www.aliexpress.com/item/1005006953997214.html)   
[CWWK q670 Reddit discussion](https://www.reddit.com/r/homelab/comments/1gmf67u/cwwk_q670_8bay_new_model_white/)   
[CWWK q670 review](https://nascompares.com/2024/05/24/cwwk-q670-gen-5-nas-8-bay-board-review/)    
[Official manual to simular mobo](https://matthewhill.uk/wp-content/uploads/2024/10/stonestorm-Q670-H670-manual.pdf)   
[q670 NAS build article](https://matthewhill.uk/general/cwwk-q670-low-power-intel-12-13-14-gen-nas-motherboard/)   



## CPU
The CPU choice was not that hard, as I already locked myself in intel 12/13/14th gen CPUs. 
Also CWWK (motherboard vendor maker) suggests using T models CPUs, because has lover power consumption, but non-T variants, are probably ok as well.

In the end I went with [Intel i5-13500T](https://www.intel.com/content/www/us/en/products/sku/230578/intel-core-i513500t-processor-24m-cache-up-to-4-60-ghz/specifications.html), because of very good deal.

You can go with something much, less powerful and be fine.



## PSU
You can find good PSU with this [PSU Low Idle Efficiency Database](https://docs.google.com/spreadsheets/d/1TnPx1h-nUKgq3MFzwl-OOIsuX_JSIurIq3JkFZVMUas/edit?gid=110239702#gid=110239702) created by [Wolfgang](https://www.youtube.com/@WolfgangsChannel)

I went with [Seasonic Platinum 520W](https://www.techpowerup.com/review/seasonic-ss-520fl/5.html), for no other reason that I just had it lying around, and it's pretty power efficient.

## GPU
THe Intel i5-13500T already has build in integrated GPU, but I wanted to experiment with some AV1 encoding, so my choice was pretty much only [Intel Arc A380](https://www.intel.com/content/www/us/en/products/sku/227959/intel-arc-a380-graphics/specifications.html)    

## Case   
There are many great cases for NAS builds. One of the most popular ones being [Fractal Node 304](https://www.fractal-design.com/products/cases/node/node-304/black/), which was ultimately my choice as well.

There are few reasons
* relatively cheap (can also be bought secondhand)
* easily stores 6 HDDs/SSDs
* can be moded with [200mm Fan Front for Fractal Node mode](https://www.printables.com/model/137181-200mm-fan-front-for-fractal-node-304/comments#makes) which looks freaking cool and is very quiet!
* I just like black color 


#### Alternatives
* Sagittarius 8-bay   
I would probably go with this one, If I would be starting from the start, but this case didn't exist when I bought Node304 few years back.
* Jonsbo N2
* Jonsbo N3
* Silverstone SST-DS380

## RAM
Because of no ECC support, I just went with cheap 2x 32 Gb U-DIMM DDR5.


## Disks
You can find cheap disks with [Disk Prices](diskprices.com), which filter all of the amazon site for the best HDD deals

I bought 
* 3x 8tb HDD
* 2x 1tb SSD
* 2x 500Gb NVMe


## More Mobo+CPU combos
I was planning to use mobo+cpu combo at first, but it always felt like I was doing some compromises. Either with number of SATA ports or CPU performance, so I decided to not go this route.

[N5105](https://www.aliexpress.com/item/1005006221619148.html)
or newer version [8505](https://www.aliexpress.com/item/1005007029351941.html)   
[ERYING M-ITX B560i](https://www.aliexpress.com/item/1005005379294420.html)   
[IKuaiOS mITX B760](https://www.aliexpress.com/item/1005007199180828.html)   
[N100](https://www.aliexpress.com/item/1005007589087631.html)   



# Software choices
Proxmox/lxc




#### Alternatives 
##### Truenas
##### Unraid
Unraid mi pride o mnoho menej efektivne spravny co sa tyka diskov a ZFS systemu

mozem ale mat rozne velkosti diskov v array co mne nepride dolezite 

+ unraid + zfs moze byt zaujimavy optionhttps://www.youtube.com/watch?v=2xLIOYpBQJY

##### Proxmox with docker
proxmox docker in lxc (share  gpu)
https://www.youtube.com/watch?v=TmNSDkjDTOs



+you can also make use of traefik and labels
https://erdaltoprak.com/blog/setting-up-a-local-reverse-proxy-on-proxmox-with-traefik-and-cloudflare/



##### Proxmox with nested truenas (passthrough for ZFS shares)
Decided to scratch the idea because of unnecessary complexity and the only advantage beeing the GUI for zfs shares. In the end its better to use something like Cockpit or Webmin. 

You lose nothing but a nice gui, you gain a ton of flexibility in proxmox.


https://forum.proxmox.com/threads/best-practice-for-truenas-virtualized-on-proxmox.133536/
https://forum.proxmox.com/threads/13th-gen-intel-proxmox-truenas-plex-hardware-transcoding-guide.125404/
https://www.youtube.com/watch?v=-qm-m4q8hIU


##### Talos
I ultimately decided not to go with the kubernetes route. The reason being I knew I would had to test, destroy, spawn cluster over and over and I just didn't want to risk it with my production cluster.

I'll definitely go down this rabbit hole with homelab 3.0

## Check out other posts in the series 
{% include post-series.html %}