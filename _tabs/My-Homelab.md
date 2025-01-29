---
icon: fa-solid fa-server
order: 2
---

## Intro
### Homelab 1.0
I'm selfhosting homelab for some time now, I've started with with just a laptop with a bunch of disks over USB.

If you think about it laptop is a really good choice to start hosting your own homelab
* laptops are power efficient
* usually comes with integrated (power efficient) GPU as well
* battery for UPS like power backup and soft shutdowns 
* free screen, touchpad, keyboard


### Homelab 2.0
> This is going to be just a very short overview. I could, talk and write about this stuff all day long, so let's just keep the introduction nice and simple. And if this made you interested, check out my posts [here.](https://blog.thetechcorner.sk/) 
{: .prompt-info }


#### Requirements
I'm living in a rented apartment. I couldn't go with something large,loud or power hungry. I also like to watch movies a lot, so I want to run my own plex/jellyfin and don't worry about proper format all the time so having GPU for transcoding is pretty usefull (not necessary with iGPU, but more hustle)

* CPU powerful enough to run all my containers and VM's
* Enough PCIe lanes, SATA ports
* PCIe x16 slot for GPU (for hardware encoding)
* 2.5G LAN
* Low energy consumption
* quiet operation when idle

#### Hardware overview
Setup guide here (coming soon) 
* CPU: [Intel i5-13500T](https://www.intel.com/content/www/us/en/products/sku/230578/intel-core-i513500t-processor-24m-cache-up-to-4-60-ghz/specifications.html) with [Noctua NH L9i 17xx cooler](https://noctua.at/en/nh-l9i-17xx) 
* Motherboard: [CWWK N13 q670](https://www.aliexpress.com/item/1005006953997214.html)
* PSU: [Seasonic Platinum 520W](https://www.techpowerup.com/review/seasonic-ss-520fl/5.html)
* GPU: [Intel Arc A380](https://www.intel.com/content/www/us/en/products/sku/227959/intel-arc-a380-graphics/specifications.html)
* Case: [Fractal Node 304](https://www.fractal-design.com/products/cases/node/node-304/black/) with [200mm Fan Front for Fractal Node mode](https://www.printables.com/model/137181-200mm-fan-front-for-fractal-node-304/comments#makes)
* RAM: 2x U-DIMM DDR5 (non ECC)
* Disks:  3x 8tb HDD, 2x 1tb SSD, 2x 500Gb Nvme

#### Containers (with links to guides)
##### Media apps
* *arrs* (radarr, sonarr, prowlarr, flaresolverr, notifiarr, bazarr, autobrr, qbitorrent, jelyseerr, tdarr, sabnzbd..)  
* jellyfin
* nextcloud

##### Network:
* Pfsense (firewall)
* Pi-Hole (DNS, adblocking)
* Nginx Proxy Manager (for tls)

##### Homelab management
* cockpit (45 drives hudson plugins)
* nut server (UPS management)
* docker
* grafana with influxDB

##### Other
* bookstack (favorite notes app)
* paperless-ngx (for scanning/storing documents)
* octo-print
...


#### Other HW
* Fujitsu futro s920
Running Pfsense firewall with 2.5Gib NIC

* Raspberry Pi 4B
Running OctoPrint server for my Prusa Mk3s

* [Jet-KVM](https://jetkvm.com/)
Running PI KVM for remote access

* UPS
VMs, disks, and everything in the PC word in general hates unscheduled shutdowns. There is an easy way to fix this with some battery backup!

* Unmanaged Edge switches 1Gbit, 2.5Gbit
* Ubiquity AP lite

The AP and switches were bought a long time ago, so they are just part of the rain forest.

## Topology
![img-description](/assets/img/homelab-2-0.png)


### Homelab 3.0 (Coming soon)
Kubernetes cluster running on talos nodes manged with gitops principles

* new node AMD Soft Router Ryzen 7 5600U


## Storage pools
I'm chose ZFS because it combines management of volumes, RAID arrays, and the file system. it's a "no-brainer" with rich feature set (copy-on-write, raidz (newly with expansion feature), differential snapshot backup...)    

I'm currently running 3 main pools
-3x8Tb HDD's in raid-z1 for storage (nextcloud, NAS, movies...)
-2x1TB SSD's in mirror for the virtualization and all the containers
-1x500TB nvme for transcoding

I run biweekly proxmox default S.M.A.R.T. tests and ZFS scrubs.


## Public access
Because I don't have public IP in this apartment, and I'm limited by the range of ports I can port forward (CGN), I chose to use combination of the Cloudflare zero trust tunnels and VPS as proxy forwarder for higher load apps (jellyfin..)