---
title: Visualize Qbittorrent metrics in the Grafana
date: 2026-01-05 20:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0, qBittorrent, grafana]
math: false
mermaid: false
series: homelab-2.0
---






# Introduction
If you are a little metrics junkey like I'm, I'm going to show you how you can send and visualize qbittorent metrics in the Grafana via prometheus exporter called [qbittorrent-exporter](https://github.com/martabal/qbittorrent-exporter)


![qbitorrent_metrics](/assets/img/posts/2026-01-05-Visualize-qbittorrent-metrics-in-the-grafana.md/grafana_final.png)  




## What is Qbittorrent-exporter?
Qbittorrent-exporter is a Prometheus exporter for Qbitorrent written in a GO. It's created by the [martabal](https://github.com/martabal), so all credit goes to him.


It uses your qbitorrent admin credentials to get access to qbitorrent metrics, stores them in the prometheus and with the help of Grafana and this [qbitorrent-dasbhoard](https://raw.githubusercontent.com/martabal/qbittorrent-exporter/main/grafana/dashboard.json) let's you visualize all interesting metrics of your downloaded ISO's ;D


## Run && Configure Qbittorrent-exporter
There are multiple ways to run this exporter. You can check them all [here](https://github.com/martabal/qbittorrent-exporter)


The easiest one is going to be to run it as docker container. To do that simply copy this docker compose.


Don't forget to change URL, Username and password accordingly


```bash
services:
  qbittorrent-exporter:
    image: ghcr.io/martabal/qbittorrent-exporter:latest
    container_name: qbittorrent-exporter
    environment:
      - QBITTORRENT_BASE_URL=http://192.168.1.10:8080
      - QBITTORRENT_PASSWORD='<your_password>'
      - QBITTORRENT_USERNAME=admin
    ports:
      - 8090:8090
    restart: unless-stopped
```




You can now check if the metrics are being exported properly. To do that open up a browser, input the docker container IP with 8090/metrics like so


Again change according to your info.  
[http://10.1.1.23:8090/metrics](http://10.1.1.23:8090/metrics)


## Import the exported mertics into prometheus database
### A - If you already have some prometheus instance
If you are already running some prometheus instance, you only need to adjust the config to scrape qbitorrent-exporter container

```bash
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: "qbittorrent"
    static_configs:
      - targets: ["10.1.1.23:8090"]
```

### B - If you don't have prometheus instance
If you don't have any prometheus instances, you can run one as a container as well.

To do so run write a simple docker compose with prometheus.conf mounted as volume like so

```bash
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - /opt/prometheus/prometheus.conf:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    restart: unless-stopped
```

Then simply create the config file on the host in the path we've specified 
```bash
vim /opt/prometheus/prometheus.conf
```
```bash
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: "qbittorrent"
    static_configs:
      - targets: ["10.1.1.23:8090"]
```

Change the IP to yours!


AFter that you will be left with the prometheus instance running on the port __9090__ 

## Setup new data source in the Grafana
To create new data source go to your Grafana instance -> Connections -> Data sources    
Add the docker container with Qbittorrent-exporter as a new Grafana data source with the type __PROMETHEUS__ like so


![grafana_datasource](/assets/img/posts/2026-01-05-Visualize-qbittorrent-metrics-in-the-grafana.md/3qRAQsKs2Ba2CQoE-image.png)  




## Import && Configure Qbitorrent dashboard
You can either create your own dashboard or use [already made one](https://raw.githubusercontent.com/martabal/qbittorrent-exporter/main/grafana/dashboard.json)


To import new dashboard go to your grafana instance -> Dashboards -> New -> Import and copy the content of the following URL to the "Import via dashboard JSON model" and click LOAD


```bash
#URL
https://raw.githubusercontent.com/martabal/qbittorrent-exporter/main/grafana/dashboard.json
```
![grafana_2](/assets/img/posts/2026-01-05-Visualize-qbittorrent-metrics-in-the-grafana.md/grafana2.png)


After that choose the Name, folder and most importantly the data source of prometheus instance (running on port 9090), which we have created in previous step. In our case the datasource is called "prometheus"


![grafana_3](/assets/img/posts/2026-01-05-Visualize-qbittorrent-metrics-in-the-grafana.md/grafana3.png)


As a last step we need to add variable with the datasource name to dashboard itself
Go to Grafana -> Dashboard -> qbitorrent -> Edit -> Settings -> Variables -> New variable


Select variable type: Data source
Name: DS_PROMETHEUS
Label: Prometheus


Change data source options -> Type: Prometheus (This is the datasouce name of prometheus instance so change according to your needs)


![grafana_4](/assets/img/posts/2026-01-05-Visualize-qbittorrent-metrics-in-the-grafana.md/grafana4.png)


## You should now see all of your qbittorent metrics!






## Check out other posts in the series
{% include post-series.html %}
