---
title: Extend Manifest v2 lifetime in your Chrome browser extensions
date: 2024-08-07 20:00:00 +0100
categories: [Homelab]
tags: [manifestV2, chrome]
math: false
mermaid: false
---

## Introduction
With Manifest V2 phase-out at our doorstep, I've decided to create this short guide on how to change the registry on your Windows to keep using Manifest V2 in your Chrome browser extensions, instead of Manifest V3 which is going to be (already is, depending on when you read this) a new default.

## Reason for staying on manifest V2
Simply put, new manifest V3 prevents all AdBlockers from preemptively blocking the ads and properly doing their job.

You can check this [arstechnica post](https://arstechnica.com/gadgets/2024/08/chromes-manifest-v3-and-its-changes-for-ad-blocking-are-coming-real-soon/), if you are interested in details.


## Windows - Easiest solution for
By far the easiest solution is to create regex file and just double click it, which will rewrite the registry for you.

To do that:
1. Open an editor of your choice (vscode, notepad..)
2. Paste the following
```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Google\Chrome]
"ExtensionManifestV2Availability"=dword:00000002
```
3. Save it as "manifestv3.reg" (careful about .txt extension that you get from notepad)
4. Double click the file
5. Profit


## Windows - Manual method
1. Open a registry editor. You can do this by opening the "run" utility with "Windows Key + R" and type in "Regedit"
2. Open the folder in path "HKEY_LOCAL_MACHINE\SOFTWARE\Policies" and create the directory "Google"
3. Create another directory inside "Google" called "Chrome"
4. Right click inside and create new file with type "DWORD (32-bit Value)" and name it "ExtensionManifestV2Availability"
5. Double-click on the file and give it a value of hexadecimal "2"
6. Restart your Chrome browser
7. Profit


## Check if you are using manifest V2
1. Open up a Chrome browser
2. In the search, type in "chrome://policy" which open up a policy window.
3. If you can see "ExtensionManifestV2Availability" with value "2" , the policy we just set is working, and you are using Manifest V2.


## Disclaimer
This solution is going to work just for some time. After that, you can just switch to Firefox, which is not planning on deprecating V2 anytime soon!