---
title: Quick fix AlmaLinux9 cloudInit kernel panic
date: 2023-06-23 11:00:00 +0100
categories: [Sysadmin]
tags: [quickfix, sysadmin, rhel9, cloudinit]
math: false
mermaid: false
---


### If you tried booting cloudinit RHEL 9 (almalinux 9, rocky 9...) and you recieved kernel panic 

![img-description](/assets/img/posts/2023-06-23-Quick-fix-AlmaLinux9-cloud-init-kernel-panic.md/kKWimage.png)



### Cause

The problem is that El9 require x86-64-v2 support and certain CPUs (modes in hypervisors) may not be able to compile with it.

https://forums.rockylinux.org/t/el9-will-require-x86-64-v2-support/5311

### Fix

> On hypervisor like Proxmox change CPU mode from kvm64 to host or any other that supports x86-64-v2 feature-set.
{: .prompt-info }

![img-description](/assets/img/posts/2023-06-23-Quick-fix-AlmaLinux9-cloud-init-kernel-panic.md/S83image.png)








