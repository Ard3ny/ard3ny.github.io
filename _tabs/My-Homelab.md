---
icon: fa-solid fa-server
order: 2
---

## Intro
I've recently upgraded my homelab setup from just a laptop with a bunch of disks to a more proper system.
While I was at it, I decided to document everything and share my ideas and maybe what I'd learned from it.


> This is going to be just a very short description. I could probably talk and write about this stuff all day long, so let's just keep the introduction nice and simple. And if this made you interested, check out my posts [here.](https://blog.thetechcorner.sk/)
{: .prompt-info }



## My requirements
I'm living in a rented apartment. I couldn't go with something large,loud or power hungry.


### My goal was to end up with a device that has all these features:
* CPU powerful enough to run all my containers and VM's
The CPU doesn't have to have a high clock speed, but it's going to need a bunch of cores for multitasking.

* Enough PCIE lanes, ports for NAS
Sometimes there is a problem with how much lines you can work with and you can have problems with not all disks registering or running at full speed.
Another thing to consider is if you want to have also integrated GPU, that thing also take some of the PCI lanes and manufactures usually mislabels this, so just google your way through all forums about the particular CPU/mobo

* Multiple LAN ports for virtualizing my network
I'm sure we all hate that router, which ISP provides you with. To fix this we can just virtualize OPNsense and pass-through NIC to it and profit!

* Low energy consumption
This one is pretty simple. The today's hardware just so good and electricity so high that you can just run some low power efficient CPU and still has no problem running everything

* GPU for hardware encoding
I also like to watch movies a lot, so I want to run my own plex/jellyfin and don't worry about proper format all the time so this one is must!.



## What I've ended up with?

* 1 mini pc (AMD Soft Router Ryzen 7 5600U) in Node 304 chassis
Running truenas as hypervisor
#### VM's:
-proxmox\
-OPNsense
#### Containers:
-*arrs*\
-nextcloud\
-jellyfin\
-Pi Hole\
-traefik\
...

* Raspberry Pi 4B
Running PI KVM for remote service, if something goes wrong. Which it usually doesn't, but you know, just in the case. And the software is just so cool I've to use it right?!

* UPS
Truenas, disks, and everything in the PC word in general hates unscheduled shutdowns. There is an easy way to fix this with some battery backup!
A bonus to this setup is that the device is so low energy demanding it can run on UPS for almost another hour (well depending on how big of a UPS you have of course)

* 5 ports Edge switch 1Gb
* Ubiquity AP lite

The AP and switch were bought a long time ago, so they are just part of the rain forest.



My whole lab infrastructure is centered around Chinese MiniPC with AMD 5600U (originally notebook processor) running Truenas Scale as a hypervisor with ZFS storage underneath.
This device is virtualizing pretty much all my services, hosting my network and is used as a NAS as well.


This device is perfect for my needs because it checks all of my requirements. It has integrated GPU, 3x M2 slots which I expanded with M2 to 5 SATA


## Topology

![img-description](/assets/img/topology2.jpg)



## Storage pools
Because I'm running ZFS I can do cool stuff like: create raids, backup with snapshots..

I'm currently running 2 main pools
-2x8Tb HDD's in raid1 (mirror) for storage (nextcloud, NAS, movies...)
-2x1TB SSD's in raid1 for the virtualization and all the truenas containers

As HDD health check I run weekly long S.M.A.R.T. tests and short tests every 2 days or so. 

For system integrity I'm running weekly ZFS scrubs as well.


In the future, I'm planning on adding more HDDs to the storage, so I'll have to switch to raidz1. Right now raidZ expansion is still in the beta, but with a few tricks it's still possible to do it. Or I'll just wait it out and a year or so the feature is going to be added to truenas scale.




## Public access
Because I don't and can't have public IP in this apartment, and I'm limited by the range of ports I can port forward (because of IPS rules) and the security issues with exposing your public IP, the Cloudflare zero trust tunnels in combination with tailscale is my go to option.

I'm only port forwarding some of my game servers for better latency.


## Conclusion
Here I end my humble homelab tour. I'm going to be posting about everything I've mentioned and linking it here. I also plan on adding more hardware and services in to the future and I'll definitely document it as well, so if you are interested, check out my blog from time to time!
