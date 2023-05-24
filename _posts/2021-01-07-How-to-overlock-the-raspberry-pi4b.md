---
title: How to overlock the raspberry PI 4b
date: 2021-01-07 10:00:00 +0100
categories: [Homelab]
tags: [raspberry]
math: false
mermaid: false
---

Raspberry PI 4 B comes with the 1.5GHz 4 cores ARM Cortex-A72 processor. But this number can be pushed even further with some easy overclocking adjustments.

This can help a lot with your future projects which might require a lot computing power.

I HIGHLY recommend using some sort of cooling case or fan because Raspberry PI 4 B has gotten so powerfull that now you pretty much need to have a fan to be able to use it to full potential.


>You might see some crashes if you don’t have a fan or you want to push your PI even harder. This might corrupt your SD card, so if you have any data that you don’t want to lose you should backup your PI or use clean Raspbian installation.
{: .prompt-warning }


## Useful information

Although Raspberry PI 4 B can run at 1.5GHz, it typically idles at 600MHz and only boost clock higher when needed.

if you don’t know what overclocking is, basically it’s just process in which we set higher allowed maximum clock speed.

Every unit is different! My settings might end up crashing your PI and vice versa. Also it should be noted that I’m using both fan and aluminium case which can absorb a lot of heat.

## Let’s get started

1. If you didn’t setup your PI yet, here is a tutorial for that.
2. After that, make sure you are running the latest update and the latest version of the Raspbian OS. For that open terminal and type
```
sudo apt update && sudo apt dist-upgrade -y
sudo shutdown -r +0
```

3. Next we need to watch our current clock speed. For that open another terminal (in a PuTTy you can just right click>Duplicate Session) and type:
```
watch -n 1 vcgencmd measure_clock arm
```
As you can see our clock speed is idling around 600MHz. And if I open something in the background (browser, for example) It changes to 1.5GHz

![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/image-2222.png)


![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/image-2233333.png)

4. We are going to set a new value for our frequency in a config file. For that we need to open a new terminal and type:
```
sudo nano /boot/config.txt
```
You should be in the config.txt file. Scroll down a little bit and you should see something like this

![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/image-24.png)

Now change it to over_voltage=2 and arm_freq=1750. Over voltage is a setting that tells how much voltage should PI send into the CPU/GPU. With higher clocks you need to give a PI more juice right? The default value is 0, but you can change it anywhere between -18 and 8.

You need to be careful because with higher voltage intake there is a higher temperature and with the higher temperature your PI will automatically slow down to prevent melting.

On the other side if you would see a yellow lightings bolt looking like this

![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/ezgif-4-beb0a7cb95d5-150x150.png
)

then your PI is not getting enough voltage or amps (in this case you would need a stronger power supply)

![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/image-25.png)

If you do have a fan, you can probably push it even more. You can see my settings here:

![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/image-26.png)

Press ctrl + o to save changed edited file and confirm with ENTER. Then press ctrl + x to exit from the file.

Now we need to reboot PI to apply the new settings. Type:
```
sudo shutdown -r +0
```

I would not try to overclock more if your PI seems stable and you are not getting any crashes.

You should play with the value on your own and try all possible combinations to fit your best preferences.

You want a faster but louder PI, push it to the maximum! Or maybe you don’t want to hear that annoying fan and you only want to use PI to learn some Linux basics. In that case maybe don’t overclock at all. It’s enterily up to you.

For last but not least I will show you how to overclock the GPU. This can be very useful if you are planning on playing games, or watching 4K content or maybe using your GPU for AI rendering.

Again open /boot/config.txt file and add a new line gpu_freq=750.

![](/assets/img/posts/2021-01-07-How-to-overlock-the-raspberry-pi4b.md/image-27.png)

Save and exit from the file, reboot the PI and you are ready to go!

I hope you didn’t overcook your PI! And if so, then just write a new image on the SD card and repeat with lower settings.
