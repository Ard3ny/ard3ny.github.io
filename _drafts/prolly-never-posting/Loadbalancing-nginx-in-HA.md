---
title: Loadbalancing Nginx in HA
date: 2023-05-29 11:03:29 +0100
categories: [Homelab, Networking]
tags: [nginx, loadbalancer, ha,]
math: false
mermaid: false
---


If you ever run an application or web server, you probably come across terms like load balancing, or high availability, failover etc.

These techniques are all concepts related to improving the reliability and performance of computer systems.

- **Load balancing** is the practice of distributing incoming network traffic across multiple servers to prevent any one server from being overwhelmed
- **High availability** refers to a system or application that is designed to be continuously available, with little to no downtime. This is typically achieved by deploying redundant components that can take over if the primary component fails
- **Failover** is a specific type of high availability strategy that involves automatically switching to a backup system when the primary system fails.

Let’s simulate a scenario where we are running 2 or 3 or more proxmox nodes in a cluster. But if all of yours application/webserver is running on one the node that’s for some reason get shutdown. Or you are running your Reverse proxy on one node and webserver on another, this still gives you ZERO high availability because, if you lose either one the whole chains collapses.

There are many ways to fix this (the best probably being Kubernetes cluster), but if you want something a little less complicated you can follow this tutorial.

The way this is setup is that we are running 2 High availability nodes (which are sharing 1 virtual IP address) and they load balance all of the traffic between 2 Reverse proxy VM’s.

![](/assets/img/wordpress_mig/04/Untitled-Diagram.drawio-1.png)


You may be asking well how the HighAvaiability nodes loadbalance traffic? There are many load balancing techniques like

- **Least Connections:** Requests are sent to the server with the least number of active connections.
- **Least Response Time:** Requests are sent to the server with the least number of active connections and the lowest average response time.
- **Round Robin:** Requests are sent to available servers sequentially, e.g., Server 1 gets Request 1, Server 2 gets Request 2, and so on.
- **IP Hash:** Requests are sent to servers based on the client’s IP address.
- **Hash:** Requests are sent to servers based on your selected key, e.g., request URL.
- **Random:** Requests are sent to any random server. Often used in conjunction with either the Least Connections and Least Response Time algorithms.

But the most used one and best suited for us is **Round Robin**, which will sends

- the first request to ReverseProxy#01
- second to ReverseProxy#02
- third to ReverseProxy#01
- …

Req:

-4VMs
![](/assets/img/wordpress_mig/04/image-1.png)

Nginx Reverse Proxies

192.168.100.80/24 vIP

192.168.100.81/24

192.168.100.82/24

HA’s

192.168.100.90/24 vIP

192.168.100.91/24

192.168.100.92/24

## Create website to test on

Reverse proxie01 and 02

apt-get install nginx -y

systemctl enable –now nginx

Check it in the browser

![](/assets/img/wordpress_mig/04/image-19.png)

## HA NODES: Install keepalived, haproxy on both servers

dnf install keepalived -y

apt-get install keepalived -y

vim /etc/keepalived/keepalived.conf

Dont forget to change priorities

```bash
#HA01
vrrp_script chk_haproxy {
  script " killall -0 haproxy"  # check the haproxy process
  interval 2 # every 2 seconds
  weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
  interface eth0 # interface to monitor
  state MASTER # MASTER on ha1, BACKUP on ha2
  virtual_router_id 51
  priority 102 # 102 on ha1, 101 on ha2...higher number=higher priority
  advert_int 1
  virtual_ipaddress {
    192.168.100.90 # virtual ip address which they will share
  }
  track_script {
    chk_haproxy
  }
}
```


```bash
#HA02
vrrp_script chk_haproxy {
  script " killall -0 haproxy"  # check the haproxy process
  interval 2 # every 2 seconds
  weight 2 # add 2 points if OK
}

vrrp_instance VI_1 {
  interface eth0 # interface to monitor
  state BACKUP # MASTER on ha1, BACKUP on ha2
  virtual_router_id 51
  priority 101 # 102 on ha1, 101 on ha2
  advert_int 1
  virtual_ipaddress {
    192.168.100.90 # virtual ip address which they will share
  }
  track_script {
    chk_haproxy
  }
}
```


#do on both servers

TO test is try ssh onto virtual IP

ssh root@192.168.100.90

and this will connects you to ha01

![](/assets/img/wordpress_mig/04/image-8.png)
![](blob:https://thetechcorner.sk/112af1f8-2c69-4071-b54c-4061fbde63c3)
Then shutdown the VM or disconnect the network interface and try again

![](/assets/img/wordpress_mig/04/image-10.png)
![](/assets/img/wordpress_mig/04/image-11.png)

# HA NODES: install haproxy (on both servers)

dnf install haproxy -y  
apt-get install haproxy -y

vim /etc/haproxy/haproxy.cfg

```bash
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    nbproc 1 
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    nbthread 2
    cpu-map auto:1/1-2 0-1
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option                  redispatch
    option http-server-close
    #option forwardfor
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend http-in
    bind *:80
    mode tcp
    option tcplog
    default_backend http-servers

frontend https-in
    bind *:443
    mode tcp
    option tcplog
    default_backend https-servers

    #you can also configure pop,imap,active-sync ports for mails   


frontend submission-in
    bind *:587
    mode tcp
    no option http-server-close
    default_backend submission-servers

frontend imap-in
    bind *:143
    mode tcp
    no option http-server-close
    default_backend imap-servers

frontend imaps-in
    bind *:993
    mode tcp
    no option http-server-close
    default_backend imaps-servers


#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend http-servers
    balance roundrobin
    mode tcp
    server reverseProxy01 192.168.100.81:80 check send-proxy #real IP of nginx1
    server reverseProxy02 192.168.100.82:80 check send-proxy #real ip of nginx2
#
backend https-servers
    balance roundrobin
    mode tcp
    server reverseProxy01 192.168.100.81:443 check send-proxy  #real IP of nginx1
    server reverseProxy02 192.168.100.82:443 check send-proxy  #real IP of nginx2

 #you can also configure pop,imap,active-sync ports for mails   

backend submission-servers
    mode tcp
    no option http-server-close
    balance roundrobin
    server mail02 192.168.100.82:587 check  #ip of smtp mail server

backend imap-servers
    mode tcp
    no option http-server-close
    balance roundrobin
    server mail02 192.168.100.82:143 check

backend imaps-servers
    mode tcp
    no option http-server-close
    balance roundrobin
    server mail02 192.168.100.82:993 check


#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
listen stats # Define a listen section called "stats"
    bind :9000 # Listen on localhost:9000
    mode http
    stats enable  # Enable stats page
    stats hide-version  # Hide HAProxy version
    stats realm test-hap01\ Statistics  # Title text for popup window
    stats uri /haproxy_stats  # Stats URI
    stats auth username:password  # Authentication credentials
```


Change IP’s, names and passowrd credentials at bottom your needs

Start the service on both VMs

## HA node: install lsync on master node

dnf install lsyncd

apt-get install lsyncd

vim /etc/lsyncd.conf

mkdir /etc/lsyncd  
vim /etc/lsyncd/lsyncd.conf.lua #pre debian aj zlozku a file musi byt lua

Change IP now its 192.168.100.9x and use 91 and 92

```bash
----
-- User configuration file for lsyncd.
--
-- Simple example for default rsync, but executing moves through on the target.
--
-- For more examples, see /usr/share/doc/lsyncd*/examples/
-- 
settings {
logfile = "/var/log/lsyncd/lsyncd.log",
statusFile = "/var/log/lsyncd/lsyncd-status.log",
statusInterval = 1
}

-- IP adress of the other node! But not the virtual one!
targetList = {
        "192.168.100.9x", 
}

for _, servers in ipairs(targetList) do
sync{	default.rsyncssh,
	source="/etc/haproxy/", 
	host=servers, 
	targetdir="/etc/haproxy/",
	delay=0,
--	delete="running",
	rsync     = {
		protect_args  = false,
		owner = true,
		perms = true
	}
}
end
```


Now create keys and add exchange them between both ha VMs

First generate key

ssh-keygen

![](/assets/img/wordpress_mig/04/image-4.png)
Now copy the key and add it to the authorized\_keys files of the other VM

```bash
root@ha01:~# cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL+LynctfyCB7OdlFDLEuVXsjHmdacwpDfAzSHUUeUrzFM/u4TugNsX3YprweelKFCjrPPFpPOS+RNyIKO+7pgFwlzheZD3XnsHOWiQDvhUsAqbrqtybvq5qTSqhg1Au0LMx4LZZYenPd8Q2Ii8y3F1/PK+Cw1rn3jFo5UsM8Bo/PgAgdQpJ1nWgVKVpElUOyLcxzukNSYA3RQac/077JR6obKGLR41JYV2N/vSDbVj8vbTIaU1Tnp1YJBvBHTlBTcwWqpIo2gFeupIFRhcuIPaUEgFVgrOZtYQuSXcEQ0MmSPwnRPCLrMEedxKO7ydIDsEguT8SIlmLUPlGQOZ5EWeRV3hXA5UjLqeaXq2FUV9gihAdDz3ruhyf1jTm40SFt91rWTkBosKXrfcVbrq7KFxWlS5q+S5sEHvMKQ4oT2yw7uB6NlXBRSovwxuvMMxM4zMaN7RXZ1+as8DOE5/yTQ8GlN6r9ZWLdsS2boJ+gY5M5OCe0ePCLYCIgnvDsoDw8= root@ha01
code
```


vim ~/.ssh/authorized\_keys

and paste it

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDL+LynctfyCB7OdlFDLEuVXsjHmdacwpDfAzSHUUeUrzFM/u4TugNsX3YprweelKFCjrPPFpPOS+RNyIKO+7pgFwlzheZD3XnsHOWiQDvhUsAqbrqtybvq5qTSqhg1Au0LMx4LZZYenPd8Q2Ii8y3F1/PK+Cw1rn3jFo5UsM8Bo/PgAgdQpJ1nWgVKVpElUOyLcxzukNSYA3RQac/077JR6obKGLR41JYV2N/vSDbVj8vbTIaU1Tnp1YJBvBHTlBTcwWqpIo2gFeupIFRhcuIPaUEgFVgrOZtYQuSXcEQ0MmSPwnRPCLrMEedxKO7ydIDsEguT8SIlmLUPlGQOZ5EWeRV3hXA5UjLqeaXq2FUV9gihAdDz3ruhyf1jTm40SFt91rWTkBosKXrfcVbrq7KFxWlS5q+S5sEHvMKQ4oT2yw7uB6NlXBRSovwxuvMMxM4zMaN7RXZ1+as8DOE5/yTQ8GlN6r9ZWLdsS2boJ+gY5M5OCe0ePCLYCIgnvDsoDw8= root@ha01 code
```

Now try logging in from HA01 to ha02

ssh root@192.168.100.92

![](/assets/img/wordpress_mig/04/image-5.png)
Repeat the proces for HA02

```
root@ha02:~# cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC8siSA6/EuE45VtPlWspQh+u117sxz5p73XvBl7W6ezygCIWHg/oOMxljLVOe13ycbtp86BFORTZ+lfWOy71n8MWp7SpVPpreEYDbJ7B1tktLk8lb+bhOhJiXMkWOXme21c0S3Dkmj1d0Ot3zDnx7tMOEyCPABon7h2N4u3j3INRKiKs4jE6Dl5ZYL6MzwTovd39k4wyFW+Fxrnjb1w8tSE2JiDz+jTaDzMcexmNFs82ol8rQxCXO5Ltz0QB2+RS3zjR1tnrvR0rkGCzJMJOaGa6yiJXd1NXJijdWMYW88MYDrV5GUZdQCaXx92wvqOKADU3vj3wf0nQoNWVjglCcmB+YuP4pEx+7ZbaQqezU1kXvh78U4BuA8ZMygaWElRfhYaAmo5PMxt7v+0No2twdT5C7tjcID6XuJrMa34GYZ9CCK2Ln4R+iEZVwek6cx9AqOK51zQMzrFb74N1uW0xm0dB+lBkwAAiH/hiZJkbgaPXiocUyKfWcWoMf48zh1eOs= root@ha02

```


![](/assets/img/wordpress_mig/04/image-6.png)
Start and enable the service

systemctl enable –now lsycnd

systemctl status –now lsycnd

if u are using root which you shouldnt u have to change sshd configuration  
vim /etc/ssh/sshd\_config  
PermitRootLogin yes

and dont forget to setup keys

systemctl restart sshd

try testing

![](/assets/img/wordpress_mig/04/image-7-1024x279.png)
## Nginx Reverse proxy node: install lsyncd (on the master node)

```bash
dnf install lsyncd
apt-get install lsyncd
```

vim /etc/lsyncd.conf

mkdir /etc/lsyncd  
vim /etc/lsyncd/lsyncd.conf.lua #pre debian aj zlozku a file musi byt lua

```bash
vim /etc/lsyncd

----
-- User configuration file for lsyncd.
--
-- Simple example for default rsync, but executing moves through on the target.
--
-- For more examples, see /usr/share/doc/lsyncd*/examples/
-- 
settings {
logfile = "/var/log/lsyncd/lsyncd.log",
statusFile = "/var/log/lsyncd/lsyncd-status.log",
statusInterval = 1
}

-- IP adress of the other node!
targetList = {
        "192.168.100.82",
}

for _, servers in ipairs(targetList) do
sync{	default.rsyncssh,
	source="/etc/nginx", 
	host=servers, 
	targetdir="/etc/nginx",
	delay=0,
--	delete="running",
	rsync     = {
		protect_args  = false,
		owner = true,
		perms = true
	}
}
end

for _, servers in ipairs(targetList) do
sync{   default.rsyncssh,
        source="/usr/share/nginx/html/",
        host=servers,
        targetdir="/usr/share/nginx/html/",
        delay=0,
--      delete="running",
        rsync     = {
                protect_args  = false,
                owner = true,
                perms = true
        }
}
end
```


Again we need to generate ssh keys and exchange them

ssh-keygen

```bash
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDKQjOvMZW04yoGeTRTERz0ZtEH1TRsYMFHUtuKG+g3oWVWSra0emdMeXQFZmq0PrPnPDQ8sKa7OSM4JIjdWsiZFvukH8ST9mjGn0RRkH3OaT7lenW++emBwlmEDgE4NHD9se1nL0L5mxZTn28aWhFDuNCTmyssru6v4U28T4qTnxDBfn8NfpWF2DC3H4IQ4y7s+EmqQBZqsymnqFmJJVgPp8+OYgUc9zsjTI4mt+7WyUB0OiFcY8+0cvZc/6Wa/o96tJJ0e/uzHfeFA+6JfIzvRZANl6aI2dAXgQc6a9JW22oRRGCAszeyHd9jdiIA4YSoThHP6MH9Se1WSHWyt6f5CABhul1AhRDuLs5sFRfWUhYI6ijRL6dYpPCllxG3qKunF/4l7NzxnbrPHBBVwkTyNu7h4vZkvG4mnf8nXJMuEi93X+HBvjedN4kGTU7qu0v8z30zkB5mWxsjDjTZmkvb9Zd/L4Djem5VUFPHGxytXhcPeS7chEeOaMErQ5hm3kE= root@reverseProxy01
```

root@reverseProxy02:~# systemctl restart lsyncd  
root@reverseProxy02:~# systemctl status lsyncd

if u are using root which you shouldnt u have to change sshd configuration  
vim /etc/ssh/sshd\_config  
PermitRootLogin yes

Try connecting

![](/assets/img/wordpress_mig/04/image-12.png)
Start and enable the service

systemctl enable –now lsycnd

systemctl status –now lsycnd

And try creating a file and see if its get synchronized

![](/assets/img/wordpress_mig/04/image-13-1024x81.png)
At this point we should be able to get some traffic thorugh HA01, HA02 or HAvIP (virtual IP) on port 80 to the nginx proxy behind it

![](/assets/img/wordpress_mig/04/image-14.png)
But nginx doesnt know how to properly handle the comunication yet, to do that we have to do a few changes

Go to master node

Create a new directory for your website’s files in /var/www/html:

sudo mkdir /var/www/html/example.com

Create an index.html file in the new directory with some simple content. For example:

sudo nano /var/www/html/example.com/index.html

Paste this

```bash

<html>
<head>
	<title>Welcome to Example.com</title>
</head>
<body>
	<h1>Hello, World!</h1>
	<p>Welcome to Example.com</p>
</body>
</html>

```


Create a new configuration file for your site in the /etc/nginx/sites-available directory:

sudo nano /etc/nginx/sites-available/example.com

add this

```bash
server {
    listen 80;
    listen [::]:80;

    root /var/www/html/example.com;
    index index.html;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ =404;
    }
}

```


Save and close the file.

5. Create a symbolic link to the new configuration file in the /etc/nginx/sites-enabled directory:

sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/

6. Test the Nginx configuration for syntax errors:

```
sudo nginx -t
```

If there are no errors, reload Nginx on both servers

```
sudo systemctl reload nginx
```


Delete ethe default template

rm -rf /etc/nginx/sites-enabled/default

rm -rf /etc/nginx/sites-avaiable/default

Now try going to the site

![](/assets/img/wordpress_mig/04/image-15.png)
Or curl it from localhost

![](/assets/img/wordpress_mig/04/image-16.png)
But you wouldnt get any answer if you would try it over HA proxies thats stans before the nginx Vms

To change that we need to change some settings in nginx

vim /etc/nginx/sites-available/example.com

```
server {
    listen 80 proxy_protocol;

    root /var/www/html/example.com;
    index index.html;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ =404;
	proxy_set_header Host $host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto "https";
	proxy_buffering off;
	proxy_read_timeout 300;
	proxy_send_timeout 120;
 	keepalive_timeout  5 5;
	tcp_nodelay        on;

    }
}

```

Save it

Restart nginx on both servers

Now you wont be able to see the site from the IP that we used before

![](/assets/img/wordpress_mig/04/image-17.png)
But if we configured everything correctly we should see trafic cominf from

192.168.100.90

![](/assets/img/wordpress_mig/04/image-18.png)
