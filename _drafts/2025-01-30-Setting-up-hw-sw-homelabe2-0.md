---
title: Setting up hardware && software [Homelab 2.0]
date: 2025-01-30 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0]
math: false
mermaid: false
series: homelab-2.0
---

# Introduction
After we've [successfully chose](https://blog.thetechcorner.sk/posts/Hw-Sw-choices-alternatives/) what components we want to use and what software to run, we can start building it.

## In this post I'll show you how to
### Hardware
* mode Fractal node 304 front panel to fit 200mm noctua fan
* put all components together
* connect new power button to front panel

## Software
* flash bios on Cwwk q670 plus [optional] 
* do basic BIOS configuration on Cwwk q670 plus
* install/configure proxmox hypervisor
* import/create ZFS pools (SSD pool for apps, HDD pools for NAS storage)



# How to
### Mode Fractal node 304 front panel
The original Fractal node 304 comes, with the fully covered front panel. I've found this to be extremely bad for the airflow in the case, so I decided it has to go.

Fortnutantly there is relatively easy solution (which also looks amazing!) and we can put big 200mm noctua fan in.

I'm talking about [this Node 304 front panel mod](https://www.printables.com/model/137181-200mm-fan-front-for-fractal-node-304/files) by pixelwave, which also maded its appearance in [Wolfgang NAS building video](https://www.youtube.com/watch?v=wwsgCkbogMM) 

1. step cut the hole   
   
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

Just push the power button, in the the hole and connect the cables, as you can see in the picture.

![button_wiring1](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/button_wiring1.jpg)


### Put all components together
![together0](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together0.jpg)


![together0](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together1.jpg)


![together0](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together2.jpg)


![together0](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together3.jpg)


![together0](/assets/img/posts/2025-01-30-Setting-up-hw-sw-homelabe2-0.md/together4.jpg)



### Connect new power button to front panel


### Flash bios on Cwwk q670 plus [optional] 

### Basic BIOS configuration

### Install/Configure Proxmox

### Import/Create ZFS pools 


# Conclusion





## Check out other posts in the series
{% include post-series.html %}