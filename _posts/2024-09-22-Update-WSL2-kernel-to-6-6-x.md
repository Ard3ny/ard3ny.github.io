---
title: Update WSL2 linux kernel to 6.6.x
date: 2024-09-22 20:00:00 +0100
categories: [Sysadmim]
tags: [WSL2, linux]
math: false
mermaid: false
---

## Update

> Only follow this post if you need kernel 6.6. If you plan on playing with cilium/eBPF or you don't need any specific feature from newer kernel go with 5.15.x.
{: .prompt-info }

I've created tutorial on that [here](https://blog.thetechcorner.sk/posts/Install-Cilium-under-WSL2-with-minikube/)

## Introduction
A few days ago, I wanted to update my linux kernel running under WSL to a newer 6.x version, because I saw they have already published 6.6.36.6 on their [microsoft WSL2 github](https://github.com/microsoft/WSL2-Linux-Kernel/releases) and I was running just 5.15.x.

But even after spamming update button and enabling all updates for all products I still wouldnt get 6.x. This is because it apparently takes time until they move this release to [microsoft update catalog](https://www.catalog.update.microsoft.com/Search.aspx?q=wsl) , which ends with the 5.10.102.2 version at the time of this writing.


## Disclaimer 
Due to config changes that happened in the WSL2 kernel release 6.6.x, some of the configs Docker requires were changed to be loadable modules instead of their previously built-in state. 

Github issues links [1](https://github.com/microsoft/WSL/issues/11771) [2](https://github.com/microsoft/WSL/issues/11742)

> If you don't add these changes, the docker won't work with WSL2.
{: .prompt-warning }


## How to
We can run everything in the docker container to make it clean.

### Create script with the following content

```bash
vim build.sh
```

```bash
WSL_COMMIT_REF=linux-msft-wsl-6.6.36.6
apt update && apt install -y git build-essential flex bison libssl-dev libelf-dev bc python3 cpio dwarves curl vim

mkdir src 
cd src 
git init
git remote add origin https://github.com/microsoft/WSL2-Linux-Kernel.git
git config --local gc.auto 0
git -c protocol.version=2 fetch --no-tags --prune --progress --no-recurse-submodules --depth=1 origin +${WSL_COMMIT_REF}:refs/remotes/origin/build/linux-msft-wsl-6.6.y
git checkout --progress --force -B build/linux-msft-wsl-6.6.y refs/remotes/origin/build/linux-msft-wsl-6.6.y

#download edited wsl-config with all the changes required for docker to run
curl -o /src/Microsoft/config-wsl https://blog.thetechcorner.sk/assets/text/config-wsl

# build the kernel
echo -ne '\n' | make -j$(nproc) KCONFIG_CONFIG=Microsoft/config-wsl

#copy the files from container
rm -rf .git
cp -r /src /output
```

### Run the container with the script
```bash
docker run --name wsl2-build --rm -it -v $PWD/output/:/output/ -v $PWD/build.sh:/build.sh ubuntu:22.04 bash -c "./build.sh"
```



> The credit for config-wsl goes to [@eapotapov](https://github.com/eapotapov), which kindly shared it on already mentioned github issue.
{: .prompt-info }

## Copy the image to /mnt/c
Create a directory for kernels on Windows filesystem

```bash
cd output/src
cp arch/x86/boot/bzImage /mnt/c/Users/<your-user>/bzImage
```

## Edit WSL config
Edit WSL config so it, uses our freshly compiled kernel. Keep the double slashes.

```
C:\Users\something\.wslconfig
```
```
[wsl2]
kernel=C:\\Users\\<your-user>\\bzImage
```

## Restart WSL
From the powershell of your PC shutdown the WSL
```powershell
#shutdown
wsl --shutdown
#start again
wsl
```

## Check your new kernel
```bash
uname -r
```
And you should see 6.6.x like so

```bash
6.6.36.6-microsoft-standard-WSL2+
```
