---
title: Quickfix Fix Element (Flatpack) keep crashing under Linux
date: 2023-05-01 10:00:00 +0100
categories: [Sysadmin]
tags: [quickfix, sysadmin, element, fix]
math: false
mermaid: false
---

Just a little random bug fix (workaround) for Element client running under Linux. (Fedora 37/38 in my case).

I’m currently running latest version 1.11.30 (flatpack version).

![img-description](/assets/img/posts/2023-01-05-Fix-Element-keep-crashing.md/image-22.png)

The problem seems to be caused, by a feature called “Message search“. Searching through messages in encrypted rooms, seems to be corrupting a database, which is causing crashes.

To workaround this problem, simply disable the feature:

### Security and Privacy --> Message Search --> Message search --> Manage --> Disable

![img-description](/assets/img/posts/2023-01-05-Fix-Element-keep-crashing.md/image-20.png)

![img-description](/assets/img/posts/2023-01-05-Fix-Element-keep-crashing.md/image-21.png)

Seems like a trivial tip, but I spend a few hours trying to find a cause and fix (not really a fix I know), so hopefully this will help you, and you can spend that time more productively.