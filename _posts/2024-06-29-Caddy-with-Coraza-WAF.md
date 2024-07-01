---
title: Caddy with Coraza WAF
date: 2024-06-29 10:00:00 +0100
categories: [Homelab, Sysadmin, dev-ops]
tags: [caddy, corazawaf, WAF]
math: false
mermaid: false
---

# Introduction
In today's digital landscape, securing web applications is more crucial than ever. With the increasing sophistication of cyber threats, integrating robust security measures is essential for protecting sensitive data and ensuring the reliability of your services.

I've wanted to create easy solution for homelab people, like myself, to integrate WAF capabilities to opensource reverse proxy like Caddy.

### What's Caddy?
[Caddy](https://github.com/caddyserver/caddy) is a modern, open-source web server written in GO. It's known for its ease of use, automatic HTTPS capabilities, and robust performance. 

It's designed to simplify web serving tasks and management of SSL/TLS certificates using Let's Encrypt.

While there are alternatives like Nginx, which is around for much longer, it demands a slightly more complex setup compared to Caddy.

### What's WAF?
The Web Application Firewall (WAF) is a security system designed to protect web applications by filtering and monitoring HTTP traffic between a web application and the Internet. It operates at the application layer (Layer 7 in the OSI model) and is used to safeguard against common web exploits such as SQL injection, cross-site scripting (XSS), and other vulnerabilities. By inspecting and filtering HTTP requests, a WAF can prevent attacks that could compromise web applications, providing an essential layer of security in the defense against cyber threats.

### What is Coraza?
[Coraza](https://github.com/corazawaf/coraza) is an open source Web Application Firewall (WAF). It is written in Go, supports ModSecurity SecLang rulesets and is 100% compatible with the OWASP Core Rule Set v4.

And what's the best, it's integrated into the [Caddy!](https://github.com/caddyserver/caddy)


### What's OWASP Core rule set v4?
The OWASP Core Rule Set (CRS) is a set of generic attack detection rules for use with web application firewalls (WAFs). Developed by the Open Web Application Security Project (OWASP), the CRS provides protection against a wide range of security threats to web applications, like

* Detect Log4j / Log4Shell
* Detect Spring4Shell
* Detect JavaScript prototype pollution
* Detect common webshells by inspecting response
* Detect path traversal in file upload
* ...


The newest version V4 (April 2022) brings new detections enchantments, improvements for false positives and [many more](https://coreruleset.org/20220428/coreruleset-v4-rc1-available/)


# Build Caddy with plugins
Easiest way to add plugins to Caddy, is building it from the source code.

To do that we can use tool called [xcaddy](https://caddyserver.com/docs/build#xcaddy)


### Install caddy
First we install default caddy and use it as template.
```bash
sudo apt-get update && sudo apt-get -y install caddy
```

### Start/enable caddy
```bash
sudo systemctl enable --now caddy
```


### Install xcaddy
```bash
#install GO dependencies
sudo apt-get update && sudo apt-get -y install golang-go 


#install xcaddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-xcaddy-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/xcaddy/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-xcaddy.list
sudo apt update
sudo apt install xcaddy
```

### Build custom Caddy with corazawaf
```bash
xcaddy build --with github.com/corazawaf/coraza-caddy/v2
```

### Add update support for custom caddy

This procedure allows us to take advantage of the default caddy configuration, systemd service files and bash-completion from the official package we have installed earlier.

```bash
sudo dpkg-divert --divert /usr/bin/caddy.default --rename /usr/bin/caddy
sudo mv ./caddy /usr/bin/caddy.custom
sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.default 10
sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.custom 50
sudo systemctl restart caddy
```

* dpkg-divert will move /usr/bin/caddy binary to /usr/bin/caddy.default and put a diversion in place in case any package want to install a file to this location.

* update-alternatives will create a symlink from the desired caddy binary to /usr/bin/caddy

* systemctl restart caddy will shut down the default version of the Caddy server and start the custom one.
 
* You can change between the custom and default caddy binaries by executing the below, and following the on screen information. Then, restart the Caddy service.

```bash
update-alternatives --config caddy
```
### Download Coraza configuration file 
```bash
mkdir -p /etc/caddy/ruleset/crs
wget https://raw.githubusercontent.com/corazawaf/coraza/main/coraza.conf-recommended -O /etc/caddy/ruleset/coraza.conf
```

> If you want to deny the malicious requests, you have to update the configuration file and change SecRuleEngine from DetectionOnly to On
{: .prompt-warning }

```bash
sudo sed -i 's/^SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/caddy/ruleset/coraza.conf
```


### Prepare OWASP Core rulesets
Coraza requires CRS (core rule sets) to function. Without these rules, Coraza would just inspect traffic without knowing what to look fore.


```bash
git clone https://github.com/coreruleset/coreruleset /tmp/coreruleset
cp /tmp/coreruleset/crs-setup.conf.example /etc/caddy/ruleset/crs/1_crs_setup.conf
```

### Host simple service to test caddy
I'm going to use plex as an enable, but you can use any service

```bash
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
echo deb https://downloads.plex.tv/repo/deb public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
sudo apt update
sudo apt install plexmediaserver
sudo systemctl enable --now plexmediaserver
```

### Prepare domain to host your service
We are going to need domain, to use caddy and use test it's WAF capabilities.

If you don't own any domains, you can use create one with [duckdns](https://www.duckdns.org/) for free.

Just register and point your DNS name to your public IP, where you plan to use Caddy as a reverse proxy.

> Make sure, you have setup firewall correctly and you have all needed ports open.
{: .prompt-info }


### Setup reverse proxy site in caddyfile
To configure caddy as reverse proxy we need to edit configuration file called "caddyfile".


```bash
vim /etc/caddy/Caddyfile
```

```bash
{
        order coraza_waf first
}

https://plex.domain.com {
        coraza_waf {
                load_owasp_crs
                directives `
                Include /etc/caddy/ruleset/coraza.conf
                Include /etc/caddy/ruleset/crs/1_crs_setup.conf
                Include @owasp_crs/*.conf
                SecRuleEngine On
                `
        }
        reverse_proxy localhost:32400
}

```

* first block specifies that we want to use coraza_waf module and it should be loaded as first 
* second block specifies our test domain name and tells caddy that we want to use coraza_waf module in this domain. We also tell caddy, where are the configuration file and CRS stored.

If you are wondering about syntax or any other functions, check out [caddy docs](https://caddyserver.com/docs/caddyfile/concepts)


#### Apply new caddyfile

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy reload --config /etc/caddy/Caddyfile
```






## Simple WAF test
For simple WAF testing we can for example, just try to manually inject some code.

Just type in your domain as usual and add code to the path.

```bash
SQL Injection: https://plex.domain.com/?id=1+and+1=2+union+select+1
XSS: https://plex.domain.com/?id=<img+src=x+onerror=alert()>
Path Traversal: plex.domain.com/?id=../../../../etc/passwd
Code Injection: plex.domain.com/?id=phpinfo();system('id')
XXE: plex.domain.com/?id=<?xml+version="1.0"?><!DOCTYPE+foo+SYSTEM+"">
```

Coraza should stop this attack and you should see HTTP ERROR 403 / ACCESS DENIED.

We can also check for this in the logs
```bash
tail /var/log/syslog
```

And as we expected, we can see an error: Access denied!

```
Jul  1 08:46:37 racknerd-1e7b85 caddy[102029]: {"level":"error","ts":1719823597.5086288,"logger":"http.handlers.waf","msg":"[client \"109.230.X.X\"] Coraza: Access denied (phase 2). SQL Injection Attack Detected via libinjection [file \"@owasp_crs/REQUEST-942-APPLICATION-ATTACK-SQLI.conf\"] [line \"5140\"] [id \"942100\"] [rev \"\"] [msg \"SQL Injection Attack Detected via libinjection\"] [data \"Matched Data: 1&1UE found within ARGS:id: 1 and 1=2 union select 1\"] [severity \"critical\"] [ver \"OWASP_CRS/4.0.0-rc1\"] [maturity \"0\"] [accuracy \"0\"] [tag \"application-multi\"] [tag \"language-multi\"] [tag \"platform-multi\"] [tag \"attack-sqli\"] [tag \"paranoia-level/1\"] [tag \"OWASP_CRS\"] [tag \"capec/1000/152/248/66\"] [tag \"PCI/6.5.2\"] [hostname \"\"] [uri \"/?id=1+and+1=2+union+select+1\"] [unique_id \"BNATCoqbGCHVFVlc\"]\n"}
Jul  1 08:46:37 racknerd-1e7b85 caddy[102029]: {"level":"error","ts":1719823597.5148249,"logger":"http.handlers.waf","msg":"[client \"109.230.13.185\"] Coraza: Access denied (phase 2). Detects MSSQL code execution and information gathering attempts [file \"@owasp_crs/REQUEST-942-APPLICATION-ATTACK-SQLI.conf\"] [line \"5239\"] [id \"942190\"] [rev \"\"] [msg \"Detects MSSQL code execution and information gathering attempts\"] [data \"Matched Data: union select found within ARGS:id: 1 and 1=2 union select 1\"] [severity \"critical\"] [ver \"OWASP_CRS/4.0.0-rc1\"] [maturity \"0\"] [accuracy \"0\"] [tag \"application-multi\"] [tag \"language-multi\"] [tag \"platform-multi\"] [tag \"attack-sqli\"] [tag \"paranoia-level/1\"] [tag \"OWASP_CRS\"] [tag \"capec/1000/152/248/66\"] [tag \"PCI/6.5.2\"] [hostname \"\"] [uri \"/?id=1+and+1=2+union+select+1\"] [unique_id \"BNATCoqbGCHVFVlc\"]\n"}
Jul  1 08:46:37 racknerd-1e7b85 caddy[102029]: {"level":"error","ts":1719823597.5178568,"logger":"http.handlers.waf","msg":"[client \"109.230.13.185\"] Coraza: Access denied (phase 2). Inbound Anomaly Score Exceeded (Total Score: 10) [file \"@owasp_crs/REQUEST-949-BLOCKING-EVALUATION.conf\"] [line \"6875\"] [id \"949110\"] [rev \"\"] [msg \"Inbound Anomaly Score Exceeded (Total Score: 10)\"] [data \"\"] [severity \"emergency\"] [ver \"OWASP_CRS/4.0.0-rc1\"] [maturity \"0\"] [accuracy \"0\"] [tag \"anomaly-evaluation\"] [hostname \"\"] [uri \"/?id=1+and+1=2+union+select+1\"] [unique_id \"BNATCoqbGCHVFVlc\"]\n"}
```

## "Deeper" WAF testing
There are many ways to test the WAF. 

People have created amazing tools to make this an incredible easy experience.

Fore example check out this amazing post from [lab.wallarm.com](https://lab.wallarm.com/test-your-waf-before-hackers/), which uses docker to deploy waf test application, with one command!

