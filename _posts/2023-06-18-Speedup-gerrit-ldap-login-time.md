---
title: Speedup Gerrit LDAP login time
date: 2023-06-18 12:00:00 +0100
categories: [Sysadmin]
tags: [sysadmin, gerrit, ldap, google-workspace]
math: false
mermaid: false
---

## Experiencing long Gerrit login times, listing members etc.? 

One of the causes can be using or switching to cloud LDAP provider, which of course takes a long time to travel and authenticate each time a request is made. And of course because "cloud is the future" let's try speeding up the process without using local LDAP provider.



## Cutting down the amount of information that is being sent

The first thing you should do is to cut on the amount of internet traffic which needs to be sent and search through

What do you get from your ldapsearch request?

```
ldapsearch -x -D "LoginName" -w Password -H ldap://127.0.0.1:1000  -b "dc=domain,dc=test" >test1 
```

```
wc -l test1 
8417 test1
```

Try to get this number as low as possible by creating smallest list of information possible (number of users, groups etc.)

In some cloud provider cases like, Google workspace, you can select which Users, groups, OUs you want to include.  

Maybe you dont need the whole company and all of the OUs, maybe you can have a group which contains all the necessary users. Start by cutting the these output informations first , which will help with the traffic and CLoud Provider (google workspace) site of things 



![img-description](/assets/img/posts/2023-06-18-Speedup-gerrit-ldap-login-time.md/ldap_connector.png)


> You can ldapsearch each time after change to see if the number is getting lower and if you still has all of the users and the groups.
{: .prompt-info }


## Change the Gerrit search-through LDAP scope 

To help the speed on the gerrit side of problem, try to define the exact OU's where users and groups are stored, pattern of name it should look for..

```
vim /etc/gerrit.config
```

```
[ldap]
        server = ldap://127.0.0.1:1636
        username =
        password = 
        accountBase = ou=Users,dc=domain,dc=test
        groupBase = ou=Groups,dc=domain,dc=test
        groupMemberPattern = (&(objectClass=groupOfNames)(member=${dn}))
        groupPattern = (&(objectClass=groupOfNames)(cn=gerrit-groups-pattern*))
```

## Cache the ldap information in the GERRIT

You can cut the information to the minimum, but you will probably still experience the unavoidable distance lag in the network time.

To fix that we can cache some of the ldap information, so it's grabbed locally most of the time.

Change times according to your needs. Try to create balance between how much time are you willing to have without users synchronization with the active directory (cloud identity...) provider and how many times a day you want someone to experience the "LDAP lag".

Meaning when the new user come, old one is deactivated or blocked, password is changed etc..   

```
vim /etc/gerrit.config
```

```
[cache "accounts"]
maxAge = 4 hour
[cache "ldap_groups"]
maxAge = 4hour
[cache "ldap_groups_byinclude"]
maxAge = 12 hour
[cache "ldap_usernames"]
maxAge = 4 hour
```

If you wanna check out what precisely these settings does, go to Gerrit documentation.

https://gerrit-review.googlesource.com/Documentation/config-gerrit.html


## Resize gerrit heapsize

Another thing you could do is to change Gerrit heap size. 

What is heap size you may ask? 
(https://www.ibm.com/docs/en/mam/7.6.0?topic=tuning-heap-size-values)

Default value should be max heapsize = 1/4 of available RAM. So to do this automatically, just increase your RAM and the heapsize will follow.

To do it manually (not recommended) change "Xmx" value of gerrit ExectStart

```
ExecStart=/usr/bin/java -Xmx2048m -jar ${GERRIT_HOME}/bin/gerrit.war daemon -d ${GERRIT_HOME}
```

