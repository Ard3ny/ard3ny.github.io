---
title: Cloudflare | Pangolin - Access your homelab from the Internet over CGNAT with help of the tunnels [Homelab 2.0]
date: 2025-04-11 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0, cloudflare, pangolin]
math: false
mermaid: false
series: homelab-2.0
---



# Introduction
Running homelab behind CGNAT no longer means you have to forgo external access or create some reverse SSH with a reverse proxy combo. 

In this post, I'll show you two modern secure tunneling solutions to bridge the gap between your private network and the public internet.


[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) - pretty well known and established for DNS, zero trust, tunnels etc..

[Pangolin](https://docs.fossorial.io/overview) - "self hosted Cloudflare tunnels" 


## Cloudflare introduction
Plenty of blog posts / videos were already made on this topic, and pretty much all the up-to-date info can be found in their [docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), so I'll just summarize the important stuff.




* Cloudflare Tunnel provides you with a secure way to connect your resources to Cloudflare without a publicly routable IP address

* With Tunnel, you do not send traffic to an external IP — instead, a lightweight daemon in your infrastructure (cloudflared) creates outbound-only connections to Cloudflare's global network

* This way, your origins can serve traffic through Cloudflare without being vulnerable to attacks that bypass Cloudflare



![cf](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/cf.png)




### CF Pros
* No Need for Port Forwarding or a static IP
* No need for self-hosting VPS (important difference, but not that big of a problem, as you will find out later)
* 5/5 Ease of Setup and Integration
* Additional Security Layers (by default you get WAF, DDoS protection...)



### CF Cons
* Require to own a domain (can get around with DuckDNS)
* Privacy Considerations (CF legally can and probably is watching your traffic/data)
* No easy TCP/UDP forwarding (for stuff like gameservers etc)




### Cloudflare tunnel setup
Again, there are so many resources on this topic that I'm just not going to repeat all of it, as I want to mainly bring something new or fill out the missing information on the web.

My favorite guides being

[Jim's Garage](https://www.youtube.com/watch?v=U8hUNw2E1ZM)   
[Christian Lempa](https://www.youtube.com/watch?v=yMmxw-DZ5Ec&t=972s)  
[DB Tech](https://www.youtube.com/watch?v=Q5dG8g4-Sx0)




And many, many more.




## Pangolin introduction

This is where the hammer really struck the nail in my opinion. I was looking for software like this for very long time. Sure, there were many solutions to CGNAT problem, but this really combines features, security, and ease of use into one package.


Pangolin is a self-hosted tunneled reverse proxy solution with access management using Gerbil && Newt (WireGuard), Traefik && Badger under the hood. All of this in very easy-to-use && configurable docker containers. 


![pangolin](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/pangolin.png)




### Pangolin Pros
* No port forwarding or static IP required (on your local network behind CGNAT)
* No privacy concerns (all data go through your app over encrypted tunnels) 
* 4/5 Ease of Setup and Integration
* Additional Security Layers (come with the option to install CrowdSec but require some work) - both pro/con
* Easy TCP/UDP forwarding (for stuff like gameservers etc)




### Pangolin Cons
* Require to own a domain (can get around with DuckDNS)
* Require self-hosting VPS with static IP
* More configuration required than using CF tunnels (still not that much)
* Additional Security Layers (come with the option to install CrowdSec but require some work) - both pro/con



As you can see, the main difference is that we NEED to have a "middleman" VPS , which will route traffic through. Sure, there are other pros/cons, but this is probably the biggest one. 


But don't worry if you don't own one!! The oracle for years provided Free Tier VPS!!! I'll show you how to do it down below.



## Pangolin setup

### Setup Free Oracle Tier VPS

Oracle may not be known for the best or most favorite company out there (hard agree), but their free tier is awesome.

This VPS comes with 

* 1x vCPU (OCPU)
* 1 GB RAM
* 0.48Gbps max network bandwidth
* 200GB block storage space
* 10TB/month outbound Data Transfer


This is plenty for just sending some traffic over.



#### How to create free tier Oracle VPS

There are plenty videos/guides on that topic and it's nothing difficult so I won't go deep on commentary.

You can check this post from [Ryan Harrison blog](https://ryanharrison.co.uk/2023/01/28/oracle-cloud-free-server.html) for a detailed guide.



* Go to [Oracle free tier site](https://www.oracle.com/uk/cloud/free/#always-free)
* Click Start for free and complete the registration.
* Once you complete the registration, in the menu, select "instances" -> Create instance
* Choose "Placement" where you want your VPS to be hosted (Frankfurt AD 2 for me)
* "Security," select default
* "Image ande Shape" select Ubuntu (or whatever you prefer)
* For "shape" select "VM.Standard.E2.1.Micro" (we dont want ARM it's more complicated)
* Paste your SSH key and you are all ready to go


#### Non-free VPS alternatives [OPTIONAL]

As I've mentioned down bellow after few weeks of testing I've found out that combination of very low CPU/RAM doesn't really work with using crowdsec. Crowdsec was just taking too much RAM from my 1GB VPS (over 30%). So If you are going with low end VPS I don't recommend it.

###### Good/cheap alternatives
* [Hetzner](https://www.hetzner.com/cloud)
Probably community most favorite VPS provider. Good price/performance tiers. Great support. Very low CPU steal. But they do a little higher prices than some cheaper, scatchier VPS providers like racknerd, contabo... (but still very very good prices)

* [RackNerd](https://www.racknerd.com/)
This one is a little scatchy, you can check them out yourself, but they tend to have high CPU steal, not great history etc... BUT they do have good deals which you can check on this [community tracker](https://racknerdtracker.com/)

So If money is an issue go with them, you should still be OK, just don't put anything important there.




#### Update VPS Security Group

By default, the network security group (NSG) will block all incoming traffic to your server apart from port port 22 (SSH).

We will add few more

* TCP 80 (http)
* TCP 443 (https)
* UDP 51820 (wireguard)
* TCP 25565 [OPTIONAL] (for TCP "gameserver" forwarding)








To change

* Go to Virtual Cloud Networks (either find it in the top bar or click over from VPS).
* In the left corner -> List scope -> Compartment -> And select your compartment (You should see your VCN)
* Click on your VCN
* In the left panel -> Security list -> Select your Security list for VPS (should be called default and should be only one)
* Click on Add Ingress Rules -> Add all the mentioned ports above according to the example below.




Example
![ingress1](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/ingress1.png)




In the end it should look like this

![ingress2](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/ingress2.png)





#### Open ports on VPS (if using virtual firewall)

In this exact case, the Ubuntu VPS from Oracle comes with an additional virtual firewall (besides the security group)




* Log in to your VPS

* To open specific ports, type

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo netfilter-persistent save
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 25565 -j ACCEPT
sudo netfilter-persistent save
sudo iptables -I INPUT 6 -m state --state NEW -p UDP --dport 51820  -j ACCEPT
sudo netfilter-persistent save
```




#### Oracle VPS "busyness" generator

This is very specific for the Oracle VPS free tier instances.

##### Explanation [source](https://www.reddit.com/r/oraclecloud/comments/122b4gf/a_simple_cron_controlled_load_generator_for/)

Because my VPS randomly stopped working or froze I investigated and find out that Oracle idles/reclaim free tier instances or straight-up stops them and sends you an email that they are out of free tier CPUs.

![oracle1](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/oracle1.png)

[Source](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)


To get around this, we can create a dummy load to simulate our VPS never being idle

```bash
 echo "*/5 * * * * root timeout 46 nice md5sum /dev/zero" | sudo tee /etc/cron.d/dummy-load 
```




* This cron runs every script every 5 minutes with the job runs for 46 seconds
* During that 46 seconds time it generates a 100% load on 1 vCPU
* The timeout command kills the job after 46 seconds, the nice command is used to decrease the priority of the job, so it should not get in the way of real work. The md5sum command is used to generate the CPU load, without using much memory or other resources. 



All credit goes to reddit user "u/eric0e". 


##### Alternative stress script [If script above didn't help]
Script and tutorial created from [TechTulege](https://www.youtube.com/watch?v=IjlZfikNKcI), but I'm going to change up values, because oracle changed 15% threshold to 20% and our instance is much smaller.

I don't like this script that much, because it put much more stress on the CPU and takes up 20% of our already small RAM space, but it's an option you can do if you don't want to switch to pay-as-you-go (read down bellow)

Test values
```bash
sudo su -
apt update
apt install supervisor stress-ng


#to put 20% of stress CPU
stress-ng --cpu 2 --cpu-load 20

#RAM dummyload
stress-ng --vm 1 --vm-bytes 20% --vm-hang 0

/etc/supervisor/conf.d/stress.conf
```

```bash
#create config
vim /etc/supervisor/conf.d/stress.conf
```

```bash
[program:cpu_stress]
command=/usr/bin/stress-ng --cpu 2 --cpu-load 20 
directory=/usr/bin/
user=root
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/stress.log

[program:memory_stress]
command=/usr/bin/stress-ng --vm 1 --vm-bytes 20%% --vm-hang 0
directory=/usr/bin/
user=root
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/var/log/stress.log
```

```bash
#Reload configuration for supervisor
supervisorctl reread
supervisorctl reload
supervisorctl status

```


##### [OPTIONAL] Change from free tier to PAYG (pay as you go) subscription

There is another trick that can help if dummy load is not enough. In my case creating dummy load helped to reduce freezes from multiple a week to once a week. But this was not enough for me.

You can switch from free tier to a pay-as-you-go subscription. But because you still uses same VM instance CPU you still won't pay anything, but your VM will much less/or never goes to idle.

* Go to the Oracle Cloud website -> Upgrade and Manage Payment -> Pay as you go Upgrade account button.

> This can be potentially dangerous, AND I DON'T RECOMMEND IT, as it can charge you WHATEVER amount. Even hundreds or thousands of eurs/dollars. Although I think it's relatively safe now, Oracle can change free tier quotas anytime. 
{: .prompt-warning }


To make it safer, we can create a budget and alert that will email us if we are going to be charged.

* Go to the Oracle Cloud website -> Budgets -> Create budget 
* Set name to "budget_alert" (or whatever you like)
* Set description to "budget alert" (or whatever you like)
* Set "Target Compartment" to the compartment where you are running your VM instance (you probably only have one)
* Set "Schedule" to "Monthly"
* Set "Budget amount" to "1Eur/1dollar"
* Set "Threshold Metric" to "Actual Spend"
* Set Threshold Type" to "Percentage of budget" 
* Set "Percentage of budget threshold" to "1%" 
* Add your email to email recipients.


### Install && Configure Pangolin (1.2.0)

> Everything here applies to version 1.2.0. To get up to date information, follow [pangolin docs](https://docs.fossorial.io/overview)
{: .prompt-info }



Installation is very simple (props to pangolin devs). 


First, we need to establish what's going to be our   
* base domain name 
* Dashboard domain name




#### Base domain name

An example of base domain could be    

```
appletest.com
```

That means EVERYTHING not declaratively stated in DNS records will be used by pangolin, because in our DNS we are going to declare

```
*.appletest.com
```



I feel like this is unnecessary, and although Pangolin states not to use any subdomains, it working perfectly fine for me, so you decide.


I'm going to use "public" subdomain for my base domain. But this can be anything, really.

```
public.appletest.com
```


And in DNS records I'll declare 

```
*.public.appletest.com
```





#### Dashboard domain name
This is where we access panel with all the information and configuration and where we create tunnels.




Depending on what you choose as your basedomain just add pangolin (or whatever you want to call it)

```
pangolin.public.appletest.com
```




### Add DNS records to your DNS provider

When you decided on domain names

* go to your DNS provider (CF in my case) and add these DNS records


![ingress2](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/dns.png)


Don't forget to change NAMES and IP to YOURS!

If you are using CF don't forget to turn off proxying (orange) cloud icon. (can cause problems)


### Continue with the install 


On our newly created free VPS (or whatever VM with public IP) type

```bash
#as root
mkdir pangolin && cd pangolin
wget -O installer "https://github.com/fosrl/pangolin/releases/download/1.2.0/installer_linux_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')" && chmod +x ./installer
sudo ./installer
```




#### For base domain 
```bash
#type
public.appletest.com
```




#### For dashboard domain name
```bash
#type
pangolin.public.appletest.com
```





#### For email
Choose the email with which you want to connect as admin to the Pangolin dashboard. 

```bash
#example
eathe@apple.com
```




#### When asked if you want to install crowdsec [Option A | Option B]
I would suggest you to "Type yes", I'll show how to properly configure it. Crowdsec is a great security tool that gives you many security benefits like

* Automated Real-Time Remediation (IP blocking or CAPTCHA challenges)
* Reduced Attack Surface (By filtering out malicious traffic using CrowdSec’s curated blocklists)   

But if you don't want to use I'll show you both options with and without it.


!!!!!! UPDATE !!!!!!!!!!
After few weeks of testing I've found out that combination of very low CPU/RAM doesn't really work with using crowdsec. Crowdsec was just taking too much RAM from my 1GB VPS (over 30%). So If you are going with low end VPS I don't recommend this combination. Either update VPS or go without crowdsec.


##### [Option A]   
YES


##### [Option B]   
No


### [Option B] - without the crowdsec
I'll show you this one first because it's easier.


#### Start the containers [Option B]

```bash
docker compose up -d 
```





And that's it. You should now have access to pangolin panel, you have chose earlier.


If the panel isn't working, the problem most likely lies in ports not beeing open or domain.


```bash
#example for me
https://pangolin.public.apple.com
```




#### Debug
For more debug check [pangolin docs](https://docs.fossorial.io/overview), their discord which you can find in their docs as well. 



Tool to check open ports   

https://www.yougetsignal.com/tools/open-ports/





### [Option A] - With the CrowdSec
#### Start the containers [Option A]

```bash
docker compose up -d 
```





#### Link CrowdSec with traefik (optional if you didn't go with crowdsec) [Option A]

UPDATE
It looks like the bottom part was automated as well. To check if you have to do bottom part check for bouncer
```bash
docker exec crowdsec cscli bouncers list
```
If you see traefik-boncer already beeing there, you don't have to create crowdsecLapiKey and manually ad it to config/traefik/dynamic_config.yml (as that is already done)


```bash
root@pangolin-public:~# docker exec crowdsec cscli bouncers list
-----------------------------------------------------------------------------
 Name             IP Address  Valid  Last API pull  Type  Version  Auth Type
-----------------------------------------------------------------------------
 traefik-bouncer              ✔️                                   api-key
-----------------------------------------------------------------------------
```


```bash
#bcs by defauly crowdsec is not configured by default we need extra steps (in 1.2.0 atleast)
#https://hhf.technology/snippets/pangolin-commands#bouncer-management
#https://github.com/fosrl/pangolin/issues/463


#First we need "bouncer"
#we have none now, to check type
docker exec crowdsec cscli bouncers list

#to add bouncer
docker exec crowdsec cscli bouncers add traefik-bouncer
#you get code like xKYHxtg54ukkgssdamk23n24k242l6fCD1nc

#add key to dynamic_config.yml 


vim config/traefik/dynamic_config.yml #relative path depending on where you have installed pangolin
#replace "crowdsecLapiKey: <add key here>" for bellow
crowdsecLapiKey: xKYHxtg54ukkgssdamk23n24k242l6fCD1nc
```




```bash
#restart docker containers
docker compose down && docker compose up -d --force-recreate 
```


#### Test if CrowdSec is working  [Option A]

```bash
#check for bouncer 
docker exec crowdsec cscli bouncers list

#add your public IP to see if you are blocked in pangolin dashboard panel
#check connection to pangolin panel
#your dashboard IP #example of mine https://pangolin.public.apple.com


docker exec -it crowdsec /bin/bash
cscli decision list
cscli decision add --ip <your-public-ip> #example cscli decision add --ip 109.212.10.173
#connection to pangolin panel should stop working


#re-enable
cscli decision remoove --ip <your-public-ip> #example cscli decision add --ip 109.212.10.173

#check metrics
docker exec -it crowdsec cscli metrics
```






#### Enable automatic CrowdSec updates  [Option A]
owdSec regularly updates list of malicious IPs etc...

```bash
#type
echo "0 * * * * root exec crowdsec cscli hub update && docker exec crowdsec cscli hub upgrade" | sudo tee /etc/cron.d/crowdsec-update-hub 
```




And that's it. You should now have access to pangolin panel, you have chose earlier.

```bash
#example for me
https://pangolin.public.apple.com
```






If the panel isn't working, the problem most likely lies in ports not being open or your domain, or CrowdSec!!

Try to fall back to option B above.




#### Debug
For more debug check [pangolin docs](https://docs.fossorial.io/overview), their discord which you can find in their docs as well. 




Tool to check open ports   

https://www.yougetsignal.com/tools/open-ports/






### Create a tunnel to you homelab in the Pangolin

* go to your pangolin dasboard site

```bash
#example for me
https://pangolin.public.apple.com
```

#### Create organization.
![organization](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/organization.png)





#### Create site
After you've successfully created orgnization

* click on "Create Site"
* Choose name "homelab" (or whatever you like)
* You will be prompted with set of credentials (as shown bellow)

![site1](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/site.png)


These credentials are used to find/authenticate from the other side of this tunnel (which his our homelab in this case).


Because I'm running proxmox I'll

* choose docker compose
* create a separate LXC container with docker installed with the help of community scripts
```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/docker.sh)"
```

* 2CPU and 2GB RAM should be plenty

* After the container is created type in the commands we got eaerlier

```bash
#example, your WILL DIFFER
vim docker-compose.yaml

services:
  newt:
    image: fosrl/newt
    container_name: newt
    restart: unless-stopped
    environment:
      - PANGOLIN_ENDPOINT=https://pangolin.public.apple.com
      - NEWT_ID=tg2zau2faqzdd2ty
      - NEWT_SECRET=ahhhgaob5jxxxwdnx2qimnm32t5ec4fv1f4h9gi8g7h46bjy
```

And start the container


```bash 
docker compose up -d 
```


After everything is done, you should see your site (tunnel) with green online status in pangolin dashboard

![site1](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/site2.png)




#### Create resource (HTTP/HTTPS)
Let's create for example tunnel to our jellyfin instance. But this can be whatever app using http/https.

![resource1](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/resource1.png)


And now point it to app local IP in your homelab.

![resource2](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/resource2.png)



#### Create resource (TCP/UDP - gameserver) 
This requires a little more manual work (alteast for now in pangolin 1.2.0). Check out their [video guide](https://www.youtube.com/watch?v=acWB5wQQoOE&t)

As previously created, create a resource and now choose TCP/UDP.

Choose whatever port you want to use, I'll go with 25565 as that is default minecraft one.


> If you go with a different port you have to add it to Oracle NSG and allow it in virtual firewall.
{: .prompt-info }



![resource3](/assets/img/posts/2025-04-09-Connect-to-your-homelab-over-CGNAT-with-help-of-tunnels-homelab-2-0.md/tcp1.png)



You will be prompted with

```bash
#Traefik: Add Entrypoints
entryPoints:
  tcp-25565:
    address: ":25565/tcp"


#Gerbil: Expose Ports in Docker Compose
ports:
  - 25565:25565
```


We need to manually add these into configs in our VPS.

```bash
vim /root/pangolin/config/traefik/traefik_config.yml   #or whatever path you have to pangolin
#add
entryPoints:
  tcp-25565:
    address: ":25565/tcp"

#under certificatesResolvers like shown bellow

....
certificatesResolvers:
  letsencrypt:
    acme:
      caServer: https://acme-v02.api.letsencrypt.org/directory
      email: blabla@gmail.com
      httpChallenge:
        entryPoint: web
      storage: /letsencrypt/acme.json
entryPoints:
  tcp-25565:
    address: ":25565/tcp"
  web:
    address: :80
  websecure:
....
```
And for the gerbil ports


```bash
vim /root/pangolin/docker-compose.yaml  #or whatever path you have to pangolin

#add
Ports:
  - 25565:25565
#under all other ports like shown bellow

  gerbil:
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    command:
      - --reachableAt=http://gerbil:3003
      - --generateAndSaveKeyTo=/var/config/key
      - --remoteConfig=http://pangolin:3001/api/v1/gerbil/get-config
      - --reportBandwidthTo=http://pangolin:3001/api/v1/gerbil/receive-bandwidth
    container_name: gerbil
    depends_on:
      pangolin:
        condition: service_healthy
    image: fosrl/gerbil:1.0.0
    ports:
      - 51820:51820/udp
      - 443:443
      - 80:80
      - 25565:25565
    restart: unless-stopped
    volumes:
      - ./config/:/var/config
```

Restart containers, and you are good to go.

```bash
docker compose down && docker compose up -d --force-recreate 
```





## Conclusion
In summary, both Cloudflare Tunnel and Pangolin offer robust solutions for accessing your homelab from behind CGNAT.

Both have their own pros/cons. Peronally I'm using both for different use cases in my holelab, but I really like where the pangolin is going, so I may switch entirely.



## Check out other posts in the series
{% include post-series.html %}