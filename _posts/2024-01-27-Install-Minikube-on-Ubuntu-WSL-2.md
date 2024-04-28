---
title: Install Minikube on Ubuntu WSL 2 
date: 2024-04-27 20:00:00 +0100
categories: [kubernetes]
tags: [kubernetes, WSL2, minukube, kubectl, helm]
math: false
mermaid: false
---
# Series Introduction
Whether you're a seasoned developer or you are just starting your Kubernetes journey as a devops engineer. I've prepared for you the new blog series, where I'll provide you with information on how to begin your Kubernetes journey. The series will get gradually harder and more interesting, but let's start with the basics.

## What is Kubernetes
Kubernetes is an open-source container orchestration platform designed to automate the deployment, scaling, and management of containerized applications. It was originally developed by Google.

It would take hours to fully explain to you what Kubernetes is or how it works. Instead, find some quality courses (I highly recommend [kodekloud] (https://kodekloud.com/learning-path/kubernetes),

After you gain some knowledge, come back to this guide to start a new project to expand your skills.


# Minikube
### What it is?
[Minikube](https://minikube.sigs.k8s.io/docs/) offers a convenient and efficient way to set up a local Kubernetes cluster. It enables simple deployment of a single or multi node Kubernetes cluster locally on your computer. It is designed to facilitate Kubernetes development and testing by providing a lightweight and easy-to-use environment.

### How does it work?
Minikube works by setting up a Kubernetes cluster on a local machine. By default, it creates a one-node cluster, but this can be easily changed, as I'll show you later.

This node creation, can be handled by different backends. These different backends are called [drivers](https://minikube.sigs.k8s.io/docs/drivers/).

List of most popular drivers:
* __Docker__ - which we will be using
* KVM2
* Virtualbox
* QEMU
* Hyper-V  
...

# Installation
### Requirements
* Virtualization enabled in BIOS
* Linux OS installed over WSL 2
* Docker

#### Install docker 
This series of commands will install all the necessary dependencies for Docker and Docker itself.

```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```

#### Setup docker
You will need to add your user to the docker group and reload settings.

```bash
sudo groupadd docker
sudo usermod -aG docker ${USER}
su -s ${USER}
```

Start docker && test if docker is working correctly
```bash
sudo service docker start
docker run hello-world
```

### Install Minikube
You can always find the newest installation guide in [minukube docs](https://minikube.sigs.k8s.io/docs/start/).

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```


### Setup Minikube
__Start the cluster__
```bash
minikube start
```

__Start the dashboard__
```bash
minikube dashboard
```

Follow the link you recieved from console. It should look something like this
```
http://127.0.0.1:[PORT]/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/#/workloads?namespace=default
```

__Move the dasboard to a backround__

By default, the dashboard is actively running on your terminal. To make it go to the background, simply press "CTRL + Z" and type the following command:
```bash
bg
```

### Install kubectl
If you are not familiar with kubectl. Kubectl is a command-line tool used to interact with Kubernetes clusters. It acts as the primary interface to control Kubernetes clusters and the workloads running on them.  It allows users to deploy and manage applications, inspect and manage cluster resources, view logs, and execute commands within containers running on Kubernetes.

__Package managers__
```bash
#brew
brew install kubectl

#debian
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
#Note: In releases older than Debian 12 and Ubuntu 22.04, folder /etc/apt/keyrings does not exist by default, and it should be created before the curl command.
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
sudo apt-get update
sudo apt-get install -y kubectl

#RHEL
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

sudo yum install -y kubectl

```

#### Test Minikube cluster with kubectl

```bash
minikube status
kubectl get po -A
```
 
### Manage Minikube cluster
#### Basic commands
```bash
#Pause/unpause the cluster
minikube pause
minikube unpause

#Start/stop the cluster
minikube stop
minukube start

#Delete cluster/s
minikube delete <cluster-name>
minikube delete -all
```

#### Install addons
[Minikube addons](https://minikube.sigs.k8s.io/docs/handbook/addons/) are optional features or functionalities that can be enabled or disabled within a Minikube cluster to enhance its capabilities or provide additional tools for development and testing.

Browse the addons 
```bash
minikube addons list
```


__Enable addon__
```bash
minikube addons enable <addon-name>
```

## Bonus
### Helm
Helm is a package manager for Kubernetes that simplifies the process of deploying, managing, and scaling applications on Kubernetes clusters. It allows users to define, install, and upgrade complex Kubernetes applications and their dependencies using reusable templates called charts.

__Installation__
```bash
#Package managers
brew install helm
#RHEL systems
sudo dnf install helm

#Debian
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

#Or install helm over script (binary)
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Check if helm is working correctly
```bash
helm version
```
