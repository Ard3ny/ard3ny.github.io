---
title: Get Keychron Launcher working on Linux (CachyOS)
date: 2026-08-08 10:00:00 +0100
categories: [Linux]
tags: [linux, keychron, cachyos]
math: false
mermaid: false
---

# Introduction
I'm guessing you are in the similar situation as I'am, when you either bought new keychron keyboard or started using linux and you are trying to open up [launcher.keychron.com](https://launcher.keychron.com) to remap some keys or change color, but you just getting "HID device connected" and you can't get past the connect page. Well I got quick solution for ya..


## Why is this happening?
The launcher is a web app that talks to your keyboard directly over WebHID. Linux has sane security measures in place, so a random userspace app (or browser tab!) can't just write to a hardware input device. We need to give ourselves permission with a custom udev rule.

## Find your keyboard
First plug in the keyboard with a cable (not bluetooth!) and check the vendor ID
```bash
lsusb | grep -i keychron

Bus 003 Device 006: ID 3434:0e20 Keychron Keychron K2 HE
```

Keychron vendor ID is `3434`, product ID will differ per model but we don't even need it.

## Create the udev rule
We are going to match the whole vendor `3434`, so this works for any Keychron model, not just mine.

```bash
echo 'KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="3434", MODE="0660", TAG+="uaccess"' | sudo tee /etc/udev/rules.d/70-keychron.rules
```

Couple of notes here
* On Arch based distros (CachyOS, EndeavourOS...) the file number needs to be lower than 73, because the `uaccess` tag is processed in `73-seat-late.rules`. Name it `99-something` like most guides tell you and it won't work!
* Thanks to `uaccess` we don't need to mess with groups or usernames, the currently logged in user gets access automatically

Now reload the rules
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

And unplug and replug the keyboard!

## Use the right browser
You probably knwo this but **Firefox doesn't support WebHID at all**, so the launcher will never connect there no matter how many udev rules you write. 

Open the launcher in any Chromium based browser instead - Chromium, Chrome, Brave, Vivaldi...

## Verify
You can check the rule actually applied by looking at the ACL of the device
```bash
getfacl /dev/hidraw0
```

You should see your user in there. Now go to [launcher.keychron.com](https://launcher.keychron.com), hit connect and you should see your keyboard!


## Credits
Credit goes to [this PSA on r/Keychron](https://www.reddit.com/r/Keychron/comments/1e5um1u/a_linux_user_psa/) and [Blaise Alleyne's writeup for Debian](https://blaise.ca/blog/2025/12/31/howto-get-the-keychron-launcher-working-in-debian-gnu-linux/) that pointed me in the right direction.