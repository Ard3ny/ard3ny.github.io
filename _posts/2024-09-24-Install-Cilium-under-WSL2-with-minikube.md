---
title: Install Cilium under WSL2 with Minikube
date: 2024-09-24 20:00:00 +0100
categories: [Sysadmim]
tags: [WSL2, linux, cilium]
math: false
mermaid: false
---

## Introduction
A few days ago, I've posted tutorial on how to compile your own linux WSL kernel which allows you to use new ones (6.x.x). 

I wanted to continue and do some testing with cilium on minikube on my localhost, but I've run in to much trouble and debug I've decided to post how to.

There are so many issues, with the kernels 6.x.x on WSL2, it's best to used 5.15.146.1 and just change a few module options while building it.

You don't really need a kernel newer than 5.15 for cilium according to their [documentation](https://docs.cilium.io/en/stable/operations/system_requirements/). So save yourself trouble and go with 5.15.x

Changes in config-wsl, which is responsible for loading the right modules.[5.15x](https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-5.15.y/arch/x86/configs/config-wsl) [6.6.x](https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-6.6.y/arch/x86/configs/config-wsl)

Docker issues on WSL2 6.x [1](https://github.com/microsoft/WSL/issues/11771) [2](https://github.com/microsoft/WSL/issues/11742) [3](https://github.com/microsoft/WSL/issues/11742) .... even when setting the right modules according to documentation



## How to
We can run everything in the docker container to make it clean.

### Create script with the following content

```bash
vim build.sh
```

```bash
WSL_COMMIT_REF=linux-msft-wsl-5.15.146.1
apt update && apt install -y git build-essential flex bison libssl-dev libelf-dev bc dwarves python3

mkdir /src
cd /src
git init
git remote add origin https://github.com/microsoft/WSL2-Linux-Kernel.git
git config --local gc.auto 0
git -c protocol.version=2 fetch --no-tags --prune --progress --no-recurse-submodules --depth=1 origin +${WSL_COMMIT_REF}:refs/remotes/origin/build/linux-msft-wsl-5.15.y
git checkout --progress --force -B build/linux-msft-wsl-5.15.y refs/remotes/origin/build/linux-msft-wsl-5.15.y

# adds support for clientIP-based session affinity
sed -i 's/# CONFIG_NETFILTER_XT_MATCH_RECENT is not set/CONFIG_NETFILTER_XT_MATCH_RECENT=y/' Microsoft/config-wsl

# Modules required by Cilium
sed -i 's/CONFIG_CRYPTO_AEAD=y/CONFIG_CRYPTO_AEAD=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_AEAD2=y/CONFIG_CRYPTO_AEAD2=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_AES=y/CONFIG_CRYPTO_AES=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_CBC=y/CONFIG_CRYPTO_CBC=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_GCM=y/CONFIG_CRYPTO_GCM=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_HMAC=y/CONFIG_CRYPTO_HMAC=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_SEQIV=y/CONFIG_CRYPTO_SEQIV=m/' Microsoft/config-wsl
sed -i 's/CONFIG_CRYPTO_SHA256=y/CONFIG_CRYPTO_SHA256=m/' Microsoft/config-wsl
echo 'CONFIG_INET_XFRM_MODE_TUNNEL=m' >> Microsoft/config-wsl
sed -i 's/CONFIG_INET_ESP=y/CONFIG_INET_ESP=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_INET_IPCOMP is not set/CONFIG_INET_IPCOMP=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_INET_XFRM_TUNNEL is not set/CONFIG_INET_XFRM_TUNNEL=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_INET6_ESP is not set/CONFIG_INET6_ESP=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_INET6_IPCOMP is not set/CONFIG_INET6_IPCOMP=m/' Microsoft/config-wsl
echo 'CONFIG_INET6_TUNNEL=m' >> Microsoft/config-wsl
echo 'CONFIG_INET6_XFRM_TUNNEL=m' >> Microsoft/config-wsl
sed -i 's/CONFIG_IP_SET_HASH_IP=y/CONFIG_IP_SET_HASH_IP=m/' Microsoft/config-wsl
sed -i 's/CONFIG_IP_SET=y/CONFIG_IP_SET=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_NET_SCH_FQ is not set/CONFIG_NET_SCH_FQ=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_NETFILTER_XT_MATCH_MARK is not set/CONFIG_NETFILTER_XT_MATCH_MARK=y/' Microsoft/config-wsl
sed -i 's/# CONFIG_NETFILTER_XT_MATCH_SOCKET is not set/CONFIG_NETFILTER_XT_MATCH_SOCKET=y/' Microsoft/config-wsl
sed -i 's/CONFIG_NETFILTER_XT_SET=y/CONFIG_NETFILTER_XT_SET=m/' Microsoft/config-wsl
sed -i 's/# CONFIG_NETFILTER_XT_TARGET_TPROXY is not set/CONFIG_NETFILTER_XT_TARGET_TPROXY=m/' Microsoft/config-wsl
sed -i 's/CONFIG_XFRM_ALGO=y/CONFIG_XFRM_ALGO=m/' Microsoft/config-wsl
echo 'CONFIG_XFRM_OFFLOAD=y' >> Microsoft/config-wsl
sed -i 's/# CONFIG_XFRM_STATISTICS is not set/CONFIG_XFRM_STATISTICS=y/' Microsoft/config-wsl
sed -i 's/CONFIG_XFRM_USER=y/CONFIG_XFRM_USER=m/' Microsoft/config-wsl

# Compile
echo -ne '\n' | make -j$(nproc) KCONFIG_CONFIG=Microsoft/config-wsl

# Copy files out of the container
rm -rf .git
cp -r /src /output
```

### Run the container with the script
```bash
docker run --name wsl2-build --rm -it -v $PWD/output/:/output/ -v $PWD/build.sh:/build.sh ubuntu:22.04 bash -c "./build.sh"
```


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
## Load the necessary modules
First you can check we don't see any loaded modules
```bash
sudo lsmod
sudo systemctl status systemd-modules-load
```

```bash
awk '(NR>1) { print $2 }' /usr/lib/modules/$(uname -r)/modules.alias | sudo tee /etc/modules-load.d/cilium.conf
sed -i 's@ConditionVirtualization=!container@#ConditionVirtualization=!container@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionDirectoryNotEmpty=|/lib/modules-load.d@#ConditionDirectoryNotEmpty=|/lib/modules-load.d@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionDirectoryNotEmpty=|/usr/lib/modules-load.d@#ConditionDirectoryNotEmpty=|/usr/lib/modules-load.d@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionDirectoryNotEmpty=|/usr/local/lib/modules-load.d@#ConditionDirectoryNotEmpty=|/usr/local/lib/modules-load.d@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionDirectoryNotEmpty=|/etc/modules-load.d@#ConditionDirectoryNotEmpty=|/etc/modules-load.d@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionDirectoryNotEmpty=|/run/modules-load.d@#ConditionDirectoryNotEmpty=|/run/modules-load.d@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionKernelCommandLine=|modules-load@#ConditionKernelCommandLine=|modules-load@' /lib/systemd/system/systemd-modules-load.service
sed -i 's@ConditionKernelCommandLine=|rd.modules-load@#ConditionKernelCommandLine=|rd.modules-load@' /lib/systemd/system/systemd-modules-load.service
sudo systemctl daemon-reload
sudo systemctl restart systemd-modules-load
sudo lsmod
```



### Install minikube

You can check how to do that in my [previous post](https://blog.thetechcorner.sk/posts/Install-Minikube-on-Ubuntu-WSL-2/)

## Run Cilium cluster
```bash
minikube start --cni=cilium
cilium status --wait
```

You should see no errors after a few minutes.