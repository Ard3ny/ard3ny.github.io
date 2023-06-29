---
title: Upgrade GAMADV-XTD3 from GAM
date: 2023-06-28 11:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, gam, gamadv-xtd3, google-workspace]
math: false
mermaid: false
---

## Disclaimer
> I've felt that GAM to GAMADV-XTD3 [upgrade tutorial](https://github.com/taers232c/GAMADV-XTD3/wiki/How-to-Upgrade-from-Standard-GAM) is unnecessary too complex, so I've created my own.
{: .prompt-info }

## What's GAM and GAMADV-XTD3?
GAM (Google Apps Manager) and GAMADV-XTD3 are free, open source command line tools for Google Workspace administrators that make managing a domain/s easier and setting up users quicker and pain-free.

GAMADV-XTD3 is rewrite/extension of GAM, but with some upgrades, and more aviable features you can use.


It allows you to do bulk actions, that would usually take a lot of time and even automate them.
Examples:
* add 100s of users at the same time
* backup/restore mail folders
* search through emails
* ...

Check out GAMADV-XTD3 [github documentation](https://github.com/taers232c/GAMADV-XTD3/wiki)



## How to upgrade from GAM?
### 1. Download and install GAMADV-ETD3 

```bash
bash <(curl -s -S -L https://raw.githubusercontent.com/taers232c/GAMADV-XTD3/master/src/gam-install.sh)
```
Follow the installation process until you see Google API inicilaization:
```bash
Type NO
```

![img-description](/assets/img/posts/2023-06-28-Upgrade-GAMADV-XTD3-from-GAM.md/cPVimage.png)


### 2. Create GAMADV-XTD3 working directory
```bash
mkdir -p /root/admin/GAMConfig
export GAMCFGDIR="/root/admin/GAMConfig"
```

### 3. Cange the alias to new GAMADV-XTD3
```bash
alias gam="/Users/admin/bin/gamadv-xtd3/gam"
source ~/.bashrc
```
### 4. Initialize GAMADV-XTD3
```bash
cd ~/bin/gamadv-xtd3
gam config drive_dir /root/admin/GAMConfig verify
```

### 5. Copy old GAM auth files into the new GAM3 directory
```bash
root@gw-utils:~/bin/gamadv-xtd3 # cp -p ~/bin/gam/client_secrets.json /root/admin/GAMConfig/
root@lgw-utils:~/bin/gamadv-xtd3 # cp -p ~/bin/gam/oauth2service.json /root/admin/GAMConfig/
root@gw-utils:~/bin/gamadv-xtd3 # cp -p ~/bin/gam/oauth2.txt /root/admin/GAMConfig/

###

root@gw-utils:~/bin/gamadv-xtd3 # ls -al /root/admin/GAMConfig/
total 28
drwxr-xr-x 3 root root 4096 Jun 15 06:40 .
drwxr-xr-x 3 root root 4096 Jun 15 06:33 ..
-rw-r--r-- 1 root root  577 May 30 08:02 client_secrets.json
-rw-r--r-- 1 root root 2817 Jun 15 06:38 gam.cfg
drwxr-xr-x 2 root root 4096 Jun 15 06:38 gamcache
-rw-r--r-- 1 root root 1544 Jun  6 12:31 oauth2.txt
-rw-rw-rw- 1 root root    0 Jun 15 06:38 oauth2.txt.lock
-rw-r--r-- 1 root root 2394 May 30 08:02 oauth2service.json
```

### 6. Update your project with/without local browser
> This is neccesarry becouse GAMADV-XTD3 uses some new APIs that GAM didnt and we need to allow them as wel.
{: .prompt-info }

```bash
gam update project

Enter your Google Workspace admin or GCP project manager email address authorized to manage project(s) gam-project-abc-123-xyz? admin@domain.com

Your browser has been opened to visit:

    https://accounts.google.com/o/oauth2/v2/auth?redirect_uri=http%3A%2F%2Flocalhost%3A8080%2F&response_type=code&client_id=...
```


You will receieve the browser link (either open it locally or paste it to another machine with browser)

If the link isnt opened localy and you will get message "unable to connect" error"

Just copy the URL of the browser with the error and paste it to terminal

```bash
#copy the following in to the browser
Enter verification code or paste "Unable to connect" URL from other computer (only URL data up to &scope required): http://127.0.0.1:8080/?state=Maa0ZUKWSlWvFIgfgfss8Hh8A0tSjMtjT0CFY&code=4/0AbUR2VN0B2n6og5TDIkxV6E24wFKUicHt3-CXt0j3NGfqKnu9bNIloGrzv8a47kURdX5DA&scope=https://www.googleapis.com/auth/cloud-platform
```

### 7. Enable GAMADV-XTD3 client access
Remoe old oath2.txt file (we need to create a new one because syntax is not the same with previous GAM)

```bash
rm -f /root/admin/GAMConfig/oauth2.txt
```

Test if it's everthing ready with
```bash
gam version
```

```bash
#The output should look like this
WARNING: Config File: /Users/admin/GAMConfig/gam.cfg, Section: DEFAULT, Item: oauth2_txt, Value: /Users/admin/GAMConfig/oauth2.txt, Not Found
GAMADV-XTD3 6.60.10 - https://github.com/taers232c/GAMADV-XTD3 - pythonsource
Ross Scroggs <ross.scroggs@gmail.com>
Python 3.10.8 64-bit final
MacOS High Sierra 10.13.6 x86_64
Path: /Users/admin/bin/gamadv-xtd3
Config File: /Users/admin/GAMConfig/gam.cfg, Section: DEFAULT, customer_id: my_customer, domain.com
```

### 8. Authorrize scopes of access
```bash
gam oauth create
```
```
Select the authorized scopes by entering a number.
Append an 'r' to grant read-only access or an 'a' to grant action-only access.

[*]  0)  Calendar API (supports readonly)
[*]  1)  Chrome Browser Cloud Management API (supports readonly)
[*]  2)  Chrome Management API - Telemetry read only
[*]  3)  Chrome Management API - read only
[*]  4)  Chrome Policy API (supports readonly)
[*]  5)  Chrome Printer Management API (supports readonly)
[*]  6)  Chrome Version History API
[*]  7)  Classroom API - Course Announcements (supports readonly)
[*]  8)  Classroom API - Course Topics (supports readonly)
[*]  9)  Classroom API - Course Work/Materials (supports readonly)
[*] 10)  Classroom API - Course Work/Submissions (supports readonly)
[*] 11)  Classroom API - Courses (supports readonly)
[*] 12)  Classroom API - Profile Emails
[*] 13)  Classroom API - Profile Photos
[*] 14)  Classroom API - Rosters (supports readonly)
[*] 15)  Classroom API - Student Guardians (supports readonly)
[ ] 16)  Cloud Channel API (supports readonly)
[*] 17)  Cloud Identity - Inbound SSO Settings (supports readonly)
[*] 18)  Cloud Identity Groups API (supports readonly)
[*] 19)  Cloud Identity OrgUnits API (supports readonly)
[*] 20)  Cloud Identity User Invitations API (supports readonly)
[ ] 21)  Cloud Storage API (Read Only, Vault/Takeout Download, Cloud Storage)
[ ] 22)  Cloud Storage API (Read/Write, Vault/Takeout Copy/Download, Cloud Storage)
[*] 23)  Contact Delegation API (supports readonly)
[*] 24)  Contacts API - Domain Shared Contacts and GAL
[*] 25)  Data Transfer API (supports readonly)
[*] 26)  Directory API - Chrome OS Devices (supports readonly)
[*] 27)  Directory API - Customers (supports readonly)
[*] 28)  Directory API - Domains (supports readonly)
[*] 29)  Directory API - Groups (supports readonly)
[*] 30)  Directory API - Mobile Devices Directory (supports readonly and action)
[*] 31)  Directory API - Organizational Units (supports readonly)
[*] 32)  Directory API - Resource Calendars (supports readonly)
[*] 33)  Directory API - Roles (supports readonly)
[*] 34)  Directory API - User Schemas (supports readonly)
[*] 35)  Directory API - User Security
[*] 36)  Directory API - Users (supports readonly)
[ ] 37)  Email Audit API
[*] 38)  Groups Migration API
[*] 39)  Groups Settings API
[*] 40)  License Manager API
[*] 41)  People API (supports readonly)
[*] 42)  People Directory API - read only
[ ] 43)  Pub / Sub API
[*] 44)  Reports API - Audit Reports
[*] 45)  Reports API - Usage Reports
[ ] 46)  Reseller API
[*] 47)  Site Verification API
[ ] 48)  Sites API
[*] 49)  Vault API (supports readonly)

     s)  Select all scopes
     u)  Unselect all scopes
     e)  Exit without changes
     c)  Continue to authorization
Please enter 0-49[a|r] or s|u|e|c:
```
Go with the default and enter your admin mail account.

```bash
#Type c to continue
c

Enter your Google Workspace admin email address? admin@domain.test
```
Paste the recieved link to the browser again.

You should see 
```bash

The authentication flow has completed.
Client OAuth2 File: /Users/admin/GAMConfig/oauth2.txt, Created

admin@server:/root/admin/bin/gamadv-xtd3$
```

### 9. Enable GAMADV-XTD3 service account access
```bash
./gam user admin@domain.test check serviceaccount
```

> If some of them fail, follow to link and try again until you see all passed and you are authorized
{: .prompt-warning }

When you see following procceed to next step.
```bash
All scopes PASSED!

Service Account Client name: 109999999999999 is fully authorized.
```

### 10. Change the default config value (

This is important in a case, if you want to run GAM in a crontab or systemcd service (without loaded environment)

```bash
vim /root/.gam/gam.cfg

#update these values
cache_dir = /root/admin/GAMConfig/gamcache
config_dir = /root/admin/GAMConfig/
```

### Finally test if everthing is working
```bash
gam info domain
```