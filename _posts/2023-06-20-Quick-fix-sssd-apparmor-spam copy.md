---
title: QuickFix SSSD apparmor spam in audit logs
date: 2023-06-20 11:00:00 +0100
categories: [Sysadmin]
tags: [quickfix, sysadmin, sssd, ldap, google-workspace]
math: false
mermaid: false
---

## FIX SSSD apparmor spam in audit logs

```bash
Jun 16 11:23:19 cloudinit-debian kernel: [  110.984314] audit: type=1400 audit(1686914599.642:668): apparmor="ALLOWED" operation="open" profile="/usr/sbin/sssd//null-/usr/libexec/sssd/s    ssd_nss" name="/proc/1335/cmdline" pid=482 comm="sssd_nss" requested_mask="r" denied_mask="r" fsuid=0 ouid=0
```


FIX

```bash
sudo ln -s /etc/apparmor.d/usr.sbin.sssd /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.sssd
systemctl restart sssd
```

### Reference
https://bgstack15.wordpress.com/2020/12/03/disable-apparmor-for-sssd/



## Cutting down the amount of information that is being sent

The first thing you should do is to cut on the amount of internet traffic that needs to be sent and search through

What do you get from your ldapsearch request?

```
ldapsearch -x -D "LoginName" -w Password -H ldap://127.0.0.1:1000  -b "dc=domain,dc=test" >test1 
```

```
wc -l test1 
8417 test1
```

Try to get this number as low as possible by creating the smallest list of information possible (number of users, groups etc.)

In some cloud provider cases like, Google Workspace, you can select which Users, groups, OUs you want to include.  

Maybe you don't need the whole company and all of the OUs, maybe you can have a group that contains all the necessary users. Start by cutting these output informations first, which will help with the traffic and CLoud Provider (google workspace) site of things 



![img-description](/assets/img/posts/2023-06-18-Speedup-gerrit-ldap-login-time.md/ldap_connector.png)


> You can ldapsearch each time after change to see if the number is getting lower and if you still has all of the users and the groups.
{: .prompt-info }



