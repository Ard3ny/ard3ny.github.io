---
title: Build a cloud-init enabled Linux VM templates on Proxmox provisioned by packer [Part 1]
date: 2023-10-14 12:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, packer, cloud-init, proxmox, templates]
math: false
mermaid: false
---
# Introduction

In the past, I've already written about cloud-init and how you can use it to it easily deploy new configured VMs.
Check out the post [here.](https://blog.thetechcorner.sk/posts/Cloud-Init-Proxmox-Templates/)

It just had one small problem. To make them up to date, from time to time, you had to manually clone, run the updates and again create templates from them.

If you wouldn't do this, then after some time update/upgrade on the VM boot would took even a few minutes, which defeats the whole purpose of making your life easier with using the cloud-init enabled images in the first place.

To fix this, I combined shell, proxmox CLI and packer to regularly run updates on the templates, so they would be ready and up to date, when you need them. I've packed this into easy to clone and re-deploy git repository.

You can the find it on my [GitHub.](https://github.com/Ard3ny/proxmox-build-template) 

# What's this project about?  

This project builds, configures and maintain cloud-init enabled Linux VMs and transform them into templates on Proxmox hypervisor.

To automate this chain, cloud-init images are used in combination with proxmox CLI and packer.


# How does it work?


1. Service executes the nightly job, which first fetches the latest cloud-init images, then it creates and configures Proxmox VMs.

2. `packer` creates templates from newly prepared VMs and configures the templates with cloud-init defaults (SSH user and public key). 

You can easily customize this and add more cloud-init defaults. 
[List of all possible defaults](https://pve.proxmox.com/wiki/Cloud-Init_Support)

If the systemd service fails for any reason, it's configured to trigger the `notify-email@%i.service`. It also sends a notification with `proxmox-mail-forward` on successful build.


# Currently used OSs

* Debian 12 (bookworm)
* AlmaLinux 9 (selinux set to permissive)
* Ubuntu 22.04.3 (Jammy)

## Installation

Installation is intended to be done on the Proxmox host itself, otherwise it won't work.

### Install dependencies
```bash
apt-get update && apt-get install libguestfs-tools wget vim git unzip
```

### Manually install packer 

Because I'm using token ID/secret as proxmox authentication method, packer must be install manually to attain newer version than proxmox currently supports as default package. New versions support this authentication method and also fixes a lot of bugs you may encounter.

https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli

TLDR 
Download the newest version (currently 201.9.4)
```bash
wget https://developer.hashicorp.com/packer/downloads#:~:text=Version%3A%201.9.4-,Download,-AMD64
```
Unzip && move the "precompile" file
```
unzip packer*
mv packer /usr/bin/
```

### Make sure these VM IDs are not used:
8999, 9000, 8000, 7999, 7000, 6999



### Clone the repository

Clone the repository to `/opt`.

```bash
git clone https://github.com/Ard3ny/proxmox-build-template.git /opt/build-template
cd /opt/build-template
```

### Create token/secret in proxmox

### Over GUI

#### In Proxmox - > Datacenter -> Permissions -> Users -> Add
![Add-User](/assets/img/posts/2023-10-14-Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer.md/add_user.png)

#### In Proxmox - > Datacenter -> Permissions -> API Tokens -> Add
![Add-Token](/assets/img/posts/2023-10-14-Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer.md/add_token.png)

> Make sure privilige separation is unchecked.
{: .prompt-warning}

When you click add you will get secret and ID info. Save those.

![Save-info](/assets/img/posts/2023-10-14-Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer.md/save_token_info.png)

#### Add permissions for the user

> To work properly user needs "PVEadmin" and "administrator" for whole / 
{: .prompt-info}

![Add-perm1](/assets/img/posts/2023-10-14-Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer.md/user_permissions.png)

![Add-perm2](/assets/img/posts/2023-10-14-Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer.md/user_permissions3.png)

### Or Over CLI (you dont have to do both)
```bash
pveum user add kubernetes@pve
pveum acl modify / -user kubernetes@pve -role Administrator
pveum acl modify / -user kubernetes@pve -role PVEAdmin
pveum user token add kubernetes@pve test_id -privsep 0
```

Complete.
 
### Configuration

Copy the environment variable files and edit them with your own parameters.

```
cp env .env && cp credentials.pkr.hcl.example credentials.pkr.hcl
vim .env
vim credentials.pkr.hcl
```

### Setup systemd timers

By default, the build-template service runs each night at 00:15.

```
make install
systemctl daemon-reload
systemctl enable --now build-template.timer
```

## Run it now (for testing)

```
/usr/bin/make -C /opt/build-template
```

# Disclaimer

I've forked this project originally created by [mfin](https://github.com/mfin/proxmox-build-template)
I've fix few bugs, added more cloud-init templates, changed install and authentication methods and extended documentation, so big "shout-out" goes to him.


# Useful links

[Packer proxmox intregration](https://developer.hashicorp.com/packer/integrations/hashicorp/proxmox/latest/components/builder/clone)
[Virt customize](https://www.libguestfs.org/virt-customize.1.html)

# To be continue

In the next post, I'll show you how you can use terraform to deploy VMs from this packer provisioned template.

