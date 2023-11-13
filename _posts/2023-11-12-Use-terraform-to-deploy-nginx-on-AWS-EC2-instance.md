---
title: Spawn and provision your VMs in proxmox with terraform [Part 2]
date: 2023-10-18 20:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, terraform, cloud-init, proxmox, templates]
math: false
mermaid: false
---
# Introduction


In this Part 2 of my mini-series on creating, maintaining and provisioning proxmox VMs, we are going to take a look on terraform and how we can use to deploy VMs and how we use infrastructure as code to automate this pipeline.


You can find part 1, in which I explain how to create and maintain template with packer and proxmox CLI [here.](https://blog.thetechcorner.sk/posts/Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer/) 




# How does it work?



1. Packer regularly creates fresh, updated, configured VM templates for us. [Part 1](https://blog.thetechcorner.sk/posts/Build-a-cloud-init-enabled-Linux-VM-templates-on-Proxmox-provisioned-by-packer/)
2. Terraform uses this template to clone and spawn new VMs as you define them.
3. Profit



## Installation


Installation can be done, on any server (it doesn't have to be proxmox host).
Just for simplifying things, I'll again do it on proxmox host itself


### Install dependencies
```
sudo apt-get install software-properties-common gnupg curl
```


### Install terraform
```
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=$(dpkg --print-architecture)] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update
sudo apt install terraform
```


### Create token/secret in proxmox (GUI version in part 1)

### TLDR Over CLI 

```
pveum user add kubernetes@pve
pveum acl modify / -user kubernetes@pve -role Administrator
pveum acl modify / -user kubernetes@pve -role PVEAdmin
pveum user token add kubernetes@pve test_id -privsep 0
```

Complete.




## Configuration


### Create env
Create working folder and copy the environment variable files and edit them with your own parameters.


```
cd ~
mkdir terraform && cd terraform
touch provider.tf vars.tf cloud-vms.tf
```



### Define the provider
A Terraform provider is responsible for understanding API interactions and exposing resources. The Proxmox provider uses the Proxmox API. This provider exposes two resources: proxmox_vm_qemu and proxmox_lxc.


From terraform documentation about [proxmox provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)


> Change parameters to your needs.
Use your proxmox IP and your token ID/secret.
{: .prompt-warning}



```
vim provider.tf
```
```
terraform {
required_providers {
proxmox = {
source = "telmate/proxmox"
version = "2.9.14"
}
}
backend "local" {
}
}


provider "proxmox" {
# url/ip of API of your proxmox host
pm_api_url = "https://10.x.x.x:8006/api2/json"
# api token id is in the form of: <username>@pam!<tokenId>
pm_api_token_id = "terraformapi@pam!test_id"
# this is the full secret wrapped in quotes.
pm_api_token_secret = "9de7c4c9-f545-4721-b4c6-e1f171aa4ab0"
pm_tls_insecure = true


# debug log
# pm_log_enable = true
# pm_log_file = "terraform-plugin-proxmox.log"
# pm_debug = true
# pm_log_levels = {
# _default = "debug"
# _capturelog = ""
# }
}
```


### Define vars
Use your variable names!
* ssh key (your user)
* name of the proxmox host node (where the clone and spawn VM accur)
* template name (of one packer created for us)
* user which you will use to login to VM
* your IP settings (static,dhcp)


Example
```
vim vars.tf
```


```
variable "ssh_key" {
default = "ssh-rsa YourSSHpubKeyXxxxxXXXXX"
}


variable "proxmox_host" {
default = "proxmox"
}
variable "template_name" {
default = "debian-ci"
}
variable "ciuser" {
default = "root"
}
variable "ipconfig" {
default = "ip=dhcp"
}


#static example
#variable "ipconfig" {
# default = "ip=10.x.x.x/24,gw=10.x.x.x"
#}
```


### Define plan


This is going to be entirely custom to your needs.
Meaning define your desired number of CPUs, RAM, name of VM, storage etc..


This plan will spawn:
* 1 VM (count=1)
* named cloud-vm-1 (name)
* on proxmox node "proxmox" (defined in vars)
* from template debian-ci (defined in vars)
* with agent enabled
* 2 CPU cores, 1024 Mb of RAM,
* ...



```
# resource "[type]" "[entity_name]"
# in our case proxmox_vm_qemu entity named cloud-initest
resource "proxmox_vm_qemu" "cloudinit-test" {
# number of VMs in entity
count = 1
# name of the new VM
name = "cloud-vm-1"


# proxmox node name in which template is stored and clone will be done
target_node = var.proxmox_host


# name of the vm/template which will be cloned
clone = var.template_name


# agent means proxmox qemu guest agent
#set agent 1 if you have it installed on template
agent = 1
os_type = "cloud-init"
cores = 2
sockets = 1
cpu = "host"
memory = 1024
scsihw = "virtio-scsi-pci"
bootdisk = "scsi0"


disk {
slot = 0
# disk size of new VM
size = "10G"
type = "scsi"
storage = "local-lvm"
iothread = 0
}


# for two NICs, just duplicate whole network section
network {
model = "virtio"
bridge = "vmbr0"
}


# Because of bug when MAC address changes the whole VM is created from scratch, with this setting tf will ignore all network changes that happens in the VM
lifecycle {
ignore_changes = [
network,
]
}




ipconfig0 = var.ipconfig
ciuser = var.ciuser
sshkeys = <<EOF
${var.ssh_key}
EOF
}
```


## Conclusion
Congratulation, you've now successfully learned to use infrastructure as a code using terraform. 


