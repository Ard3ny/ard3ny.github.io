---
title: Create proxmox cluster
date: 2023-01-22 11:03:29 +0100
categories: [Homelab]
tags: [proxmox]
math: false
mermaid: false
---

I’ve already done a tutorial on how “Change your old laptop into proxmox node“, but what do you do in case you run out of RAM or CPU cores? Or maybe you want to create some HA(high availability) in your Homelab environment.

Well, in that case, you can just add another proxmox node and cluster them together! Sure, you can have them running separately, but grouping nodes has many advantages like

[Grouping nodes](https://pve.proxmox.com/wiki/Cluster_Manager) into a cluster has the following advantages:

* Centralized, web-based management
* Multi-master clusters: each node can do all management tasks
* Use of pmxcfs, a database-driven file system, for storing configuration files, replicated in real-time on all nodes using "corosync"
* Easy migration of virtual machines and containers between physical hosts
* Fast deployment
* Cluster-wide services like firewall and HA


## Requirements
* 2 or more proxmox nodes
* same network for all of them

In a case you want to know how to create a single proxmox node, follow my previous tutorial.

## Creating a cluster
In this example, I have two running proxmox nodes (proxmox, proxmox2).

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-45.png)
Log in the first node (proxmox) and go to > Datacenter >  Cluster, and select Create Cluster.

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-46.png)

2. Select a cluster name, and click create (leave the cluster network on default). The cluster will then be created and you’ll be able to join it from other Proxmox instances.

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-47.png)

## Join a cluster
Go to node 1 in which you’ve created a cluster (in my case name is proxmox). And go to Datacenter > Cluster > Join information

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-49.png)

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-50.png)

You will need both, the Fingerprint and Join Information to join the cluster. Select Copy Information.

2. Open your second Proxmox node (proxmox2)  and select Datacenter, Cluster, and Join Cluster.
![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-51.png)

3. You will see Cluster Join window prompt like this. Simply paste the copied information and type in password and click Join.
![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-52.png)

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-53.png)

You should see a Task viewer: Join Cluster window. It may seem like it stuck, but don’t worry, I think it’s just a bug and your cluster should already be up and running.

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-54.png)

![img-description](/assets/img/posts/2023-01-22-Create-proxmox-cluster.md/image-55.png)

Now you can monitor and provision both of them from one place.

You can also migrate your "VM’s" and LXC containers between them and you can also set up something called HA (high viability) which will keep your nodes running even if one of the nodes shuts down.