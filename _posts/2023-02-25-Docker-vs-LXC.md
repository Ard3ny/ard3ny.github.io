---
title: Docker vs LXC
date: 2023-02-25 11:03:29 +0100
categories: [Homelab]
tags: [docker, proxmox]
math: false
mermaid: false
---

## What is a Virtual machine and what’s a container (VM vs VE)

A Virtual Machine (VM) is a complete software emulation of a physical machine. It allows multiple operating systems to run simultaneously on a single physical machine, each with full isolation between these OS’s.

A Virtual Environment (VE), also known as a container, is a lightweight virtualization solution that runs multiple isolated applications within a single operating system instance. Each container shares the same kernel and operating system libraries but has its own isolated file system and network stack

The main difference between VMs and VEs is that VMs provide complete hardware emulation and full isolation between multiple operating systems, while VEs share the same operating system kernel and libraries and provide isolated user space environments.

![img-description](/assets/img/posts/2023-25-02-Docker-vs-LXC/Untitled-Diagram.drawio-1.png)


## What types of containers are there?

System containers (LXD) are similar to virtual or physical machines. They run a full operating system inside them, you can run any type of workload, and you manage them exactly as you would a virtual or a physical machine. System containers are usually long-lasting and you could host several applications within a single system container. 

Application containers (for example Docker), also known as process containers, are containers that package and run a single process or a service per container. They run stateless types of workloads that are meant to be ephemeral. This means that these containers are temporary, and you can create, delete and replace containers easily as needed.

Both application and system containers share a kernel with the host operating system and utilize it to create isolated processes. The main difference is illustrated below – application containers run a single app/process, whereas system containers run a full operating system giving them flexibility for the workload types they support.

## What is Docker

Docker is an open-source containerization technology that focuses on running a single application in an isolated environment. Its Docker Engine enables you to create, run, or distribute containers.

With Docker, developers can easily create, distribute, and run applications across different environments, such as local machines, cloud servers, or Kubernetes clusters

Docker also provides a vast ecosystem of images and tools that developers can use to build and deploy their applications, including Docker Hub, a public repository of Docker images, and Docker Compose, a tool for defining and running multi-container applications.

### Key points
* is a virtual installation of an application 
* not persistent (only mounted folders)
* containers are never upgraded, only replaced (so they always have to be destroyed)
* can and often uses NAT


![img-description](/assets/img/posts/2023-25-02-Docker-vs-LXC/docker.drawio-4.png)Docker architecture


## What is LXC/LXD
### LXC – Linux containers
It’s a solution for virtualization software at the operating system level within the Linux kernel. Unlike traditional hypervisors (VMware, KVM and Hyper-V), LXC lets you run single applications in virtual environments, although you can also virtualize an entire operating system inside an LXC container if you’d like.

"LXC’s" main advantages include making it easy to control a virtual environment using tools from the host OS, requiring less overhead than a traditional hypervisor and increasing the portability of individual apps by making it possible to distribute them inside containers.

You may notice LXC is a lot like Docker container, it’s because LXC used to be the underlying technology that made Docker. More recently, however, Docker has gone in its own direction and no longer depends on LXC

### Key points

* LXC is virtual OS that can be used to install application
* LXC works like a Virtual Machine, but it’s faster and leaner
* LXC uses kernel of the host and the host services (so if host kernel is missing some feature, LXC/LXD may be a bad choice)
* LXC doesn’t virtualize the hardware
both LXC/LXD can use NAT, but its more common to use bridged to a LAN
* LXC can be upgraded without destroying original container
you have to install the entire application and all dependencies

![img-description](/assets/img/posts/2023-25-02-Docker-vs-LXC/docker.drawio-2.png)LXC Architecture


### LXD – Linux Container Daemon
LXD is a lightweight container hypervisor. It’s an extension/interface of LXC

LXD is basically a management CLI system on top of LXC

The more technical way to define LXD is to describe it as a REST API that connects to "libxlc", the LXC software library. LXD, which is written in Go, creates a system daemon that apps can access locally using a Unix socket, or over the network via HTTPS.

"LXD’s" main selling points include the following:

A host can run many LXC containers using only a single system daemon, which simplifies management and reduces overhead. With pure-play LXC, you’d need separate processes for each container.
The LXD daemon can take advantage of host-level security features to make containers more secure. On plain LXC, container security is more problematic.
Because the LXD daemon handles networking and data storage, and users can control these things from the LXD CLI interface, it simplifies the process of sharing these resources with containers.
LXD offers advanced features not available from LXC, including live container migration and the ability to snapshot a running container.

![img-description](/assets/img/posts/2023-25-02-Docker-vs-LXC/lxd.drawio-2.png)LXD Architecture


## LXC/LXD vs Docker

LXD is designed for hosting virtual environments that “will typically be long-running and based on a clean distribution image,” whereas “Docker focuses on ephemeral, stateless, minimal containers that won’t typically get upgraded or re-configured but instead just be replaced entirely.”

This means you should consider the type of deployment you will have to manage before making a choice regarding LXD or Docker. Are you going to be spinning up large numbers of containers quickly based on generic app images, If so Docker is a better choice. Alternatively, if you intend to virtualize an entire OS, or to run a persistent virtual app for a long period, LXD will likely prove a better solution.

The second factor to consider is your host environment. LXD only supports Linux (at least for now). So if your servers run some less common flavor of Linux, Windows, LXD won’t work well for you. In contrast, Docker is pretty portable across almost any Linux-based OS, and you can now even run Docker natively on Windows and OS X.

![img-description](/assets/img/posts/2023-25-02-Docker-vs-LXC/GqtGCm4.png)LXC vs docker

## Proxmox LXC vs Docker

I’m a big proxmox fan because it’s a great open source project, easy to use, ZFS supported hypervisor and use it on pretty much a daily basis in my work or my homelab I’m going to mention it’s LXC hypervisor as well.

Proxmox Containers are how we refer to containers that are created and managed using the Proxmox Container Toolkit (pct). They also target system virtualization and use LXC as the basis of the container offering.

The Proxmox Container Toolkit (pct) is tightly coupled with Proxmox VE. This means that it is aware of cluster setups, and it can use the same network and storage resources as QEMU virtual machines (VMs). You can even use the Proxmox VE firewall, create and restore backups, or manage containers using the HA framework. Everything can be controlled over the network using the Proxmox VE API.

![img-description](/assets/img/posts/2023-25-02-Docker-vs-LXC/gui-create-ct-general.png)Proxmox pct


Docker aims at running a single application in an isolated, self-contained environment. These are generally referred to as “Application Containers”, rather than “System Containers”. You manage a Docker instance from the host, using the Docker Engine command line interface. It is not recommended to run docker directly on your Proxmox VE host.

>If you want to run application containers, for example, Docker images, it is best to run them inside a Proxmox Qemu VM.
But it is also do-able inside LXC
{: .prompt-tip }

## Conclusion

My favorite combination is either to run VM and the docker images inside of it or use LXC with proxmox pct manager, but the choice is yours.

Ultimately, both Docker and LXC/LXD have their strengths and weaknesses, and the choice between the two largely depends on your specific needs and priorities. By weighing the pros and cons of each technology, you can make an informed decision and choose the best tool for your use case.