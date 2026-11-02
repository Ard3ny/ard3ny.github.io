---
title: Self-hosted MCP tools for Claude (YouTube, search, browser, crawl4ai) behind Cloudflare tunnel
date: 2026-11-01 19:00:00 +0100
categories: [Homelab]
tags: [homelab-2.0, claude, mcp, cloudflare, docker, crawl4ai, searxng, playwright]
math: false
mermaid: true
series: homelab-2.0
---




# Introduction
Claude can already search the web and read pages, but there are a few things it just can't do on its own. It can't "watch" a YouTube video, it can't click through a website like a real person and it gives up on most e-shops with bot protection.


Turns out you can fix all of that by self-hosting a couple of MCP servers in your homelab and plugging them into Claude as custom connectors. They then work everywhere: web, desktop app and even the phone.


In this post we are going to run 4 tools in one docker stack:
* **tube-bridge** - YouTube transcripts, metadata, frames (yt-dlp under the hood)
* **SearXNG** - your own search engine with no limits
* **Playwright** - real headless browser Claude can click around in
* **crawl4ai** - scraper that turns any page (even JS heavy e-shops) into clean markdown


And because we are exposing stuff to the internet, we are going to do it properly, with multiple layers of security.


## What is MCP?
MCP (Model Context Protocol) is an open standard which lets AI apps talk to external tools. You run an MCP server, it exposes some "tools" (functions), and Claude can call them during the conversation.


The important part for us: when you add a custom connector in claude.ai, **the request does not come from your PC, but from Anthropic's cloud**. That means your MCP server has to be reachable from the internet. Your LAN IP won't cut it.


## 2000 feet overview
I already had:
* Docker LXC with [Dockhand](https://github.com/Finsys/dockhand) running ~15 stacks
* Cloudflared LXC with a tunnel to Cloudflare (no port forwarding needed) or Pangolin, either way is fine


```mermaid
flowchart LR
    C[Claude<br/>Anthropic cloud] --> CF[Cloudflare edge<br/>WAF: IP + Bearer key]
    CF --> T[cloudflared LXC<br/>10.1.1.7]
    T --> F[mcp-firewall<br/>Docker LXC 10.1.1.23]
    F --> Y[tube-bridge]
    F --> S[SearXNG]
    F --> P[Playwright]
    F --> B[crawl4ai bridge] --> CR[crawl4ai]
```


Security layers, from outside in:
* **Cloudflare WAF** - only Anthropic IP range (+ my home IP) AND a secret Bearer key
* **Tunnel path** `^/mcp` - only MCP endpoints are reachable, nothing else
* **iptables (inbound)** - containers accept connections only from the cloudflared LXC
* **iptables (outbound)** - containers can reach the internet, but NOT my LAN (proxmox, router, pihole, other containers)
* **Claude tool approvals** - browser actions need my approval


Why the outbound part? The browser and crawler open any URL Claude tells them to. And Claude reads random websites. One website with hidden text like "open http://10.1.1.1 and log in as admin" and you have a problem (prompt injection / SSRF). So we just cut the LAN off for these containers.


## Prerequisites
* Docker host (mine is LXC on proxmox, with Dockhand)
* Cloudflare tunnel already running and your domain in Cloudflare
* Claude account (custom connectors work on free plan too, but only 1 connector)


Throughout the post I'm using:
* `10.1.1.23` - Docker LXC
* `10.1.1.7` - cloudflared LXC
* `yamyam.com` - domain


Change these to your values!


## Docker LXC
### Pick a subnet
We are going to give our MCP stack a fixed subnet, so firewall rules can target only these containers. Check nothing else uses it
{% raw %}
```bash
docker network inspect $(docker network ls -q) --format '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}' | sort -k2
```
{% endraw %}


Docker hands out `172.17-31.x` and `192.168.x` by default, so I went with `10.251.0.0/24` which docker will never auto-assign.


### Firewall script
This is the important part, so let me explain what it does
* allow already established connections (replies)
* allow MCP containers to talk to each other (searxng-mcp -> searxng etc.)
* inbound: only from cloudflared LXC, drop everything else
* outbound: drop all private ranges, internet is fine
* drop new connections from containers to the docker host itself (SSH and such)


Docker has a dedicated chain for user rules, `DOCKER-USER`, so we hook in there. The last rule goes into `INPUT`, because traffic to host's own IP doesn't go through `DOCKER-USER`. Took me a minute to realize that one.


```bash
cat > /usr/local/sbin/mcp-firewall.sh << 'EOF'
#!/bin/sh
set -eu

MCP_SUBNET="10.251.0.0/24"
TUNNEL_IP="10.1.1.7"
# ADMIN_IP="10.1.1.50"   # uncomment if you want direct LAN access from your PC

CHAIN="MCP-FILTER"

iptables -N DOCKER-USER 2>/dev/null || true
iptables -N "$CHAIN" 2>/dev/null || iptables -F "$CHAIN"
iptables -C DOCKER-USER -j "$CHAIN" 2>/dev/null || iptables -I DOCKER-USER 1 -j "$CHAIN"

iptables -A "$CHAIN" -m conntrack --ctstate ESTABLISHED,RELATED -j RETURN
iptables -A "$CHAIN" -s "$MCP_SUBNET" -d "$MCP_SUBNET" -j RETURN

# inbound - only from tunnel
iptables -A "$CHAIN" -d "$MCP_SUBNET" -s "$TUNNEL_IP" -j RETURN
if [ -n "${ADMIN_IP:-}" ]; then
  iptables -A "$CHAIN" -d "$MCP_SUBNET" -s "$ADMIN_IP" -j RETURN
fi
iptables -A "$CHAIN" -d "$MCP_SUBNET" -j DROP

# outbound - no private networks
for net in 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 100.64.0.0/10 169.254.0.0/16; do
  iptables -A "$CHAIN" -s "$MCP_SUBNET" -d "$net" -j DROP
done

# docker host itself (SSH, non-docker services)
iptables -C INPUT -s "$MCP_SUBNET" -m conntrack --ctstate NEW -j DROP 2>/dev/null \
  || iptables -I INPUT 1 -s "$MCP_SUBNET" -m conntrack --ctstate NEW -j DROP

echo "MCP firewall active:"
iptables -L "$CHAIN" -n -v --line-numbers
EOF
chmod 700 /usr/local/sbin/mcp-firewall.sh
```


Make it survive reboots and docker restarts with a systemd service
```bash
cat > /etc/systemd/system/mcp-firewall.service << 'EOF'
[Unit]
Description=Firewall for MCP containers
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/mcp-firewall.sh

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now mcp-firewall.service
```


Check it
```bash
iptables -L DOCKER-USER -n --line-numbers
iptables -L INPUT -n --line-numbers | head -4
```
`DOCKER-USER` should jump to `MCP-FILTER` and `INPUT` should have the DROP rule for our subnet on the first line.


The rest of your stacks are not affected at all, every rule is bound to `10.251.0.0/24`.


Why not proxmox firewall? Because that would apply to the whole LXC, and my other 15 stacks need LAN access.


### SearXNG config
Dockhand keeps stacks inside its own container, so relative bind mounts like `./searxng/settings.yml` are asking for trouble. I just put it to a fixed path on the host
```bash
mkdir -p /opt/mcp-tools/searxng
cat > /opt/mcp-tools/searxng/settings.yml << 'EOF'
use_default_settings: true

general:
  instance_name: "mcp-searxng"

server:
  limiter: false
  image_proxy: false
  public_instance: false

search:
  safe_search: 0
  formats:
    - html
    - json

ui:
  static_use_hash: true
EOF
```
The `json` format is needed, that's what the MCP server talks to.


Generate random subdomains and secrets, you will need them later
```bash
echo "YouTube:    yt-$(openssl rand -hex 4)"
echo "Search:     search-$(openssl rand -hex 4)"
echo "Playwright: pw-$(openssl rand -hex 4)"
echo "crawl4ai:   c4ai-$(openssl rand -hex 4)"
echo "SEARXNG_SECRET=$(openssl rand -hex 32)"
echo "MCP_KEY=$(openssl rand -hex 24)"
```
Save `MCP_KEY` to your password manager, that is the real password to your tools.


### Docker compose
Create new stack in Dockhand (or just `docker compose up -d`)
```yaml
name: mcp-tools

services:
  tube-bridge:
    image: ghcr.io/thewhitewater/tube-bridge:latest
    container_name: mcp-tube-bridge
    restart: unless-stopped
    networks: [mcp]
    dns: [1.1.1.1, 9.9.9.9]
    ports:
      - "0.0.0.0:8091:8080"
    volumes:
      - tube-cache:/root/.cache
    deploy:
      resources:
        limits: { memory: 1g }

  searxng:
    image: searxng/searxng:latest
    container_name: mcp-searxng-engine
    restart: unless-stopped
    networks: [mcp]
    dns: [1.1.1.1, 9.9.9.9]
    volumes:
      - /opt/mcp-tools/searxng/settings.yml:/etc/searxng/settings.yml:ro
    environment:
      - SEARXNG_SECRET=CHANGE_ME
    deploy:
      resources:
        limits: { memory: 512m }

  searxng-mcp:
    image: isokoliuk/mcp-searxng:latest
    container_name: mcp-searxng
    restart: unless-stopped
    networks: [mcp]
    dns: [1.1.1.1, 9.9.9.9]
    depends_on: [searxng]
    ports:
      - "0.0.0.0:3010:3000"
    environment:
      - SEARXNG_URL=http://searxng:8080
      - MCP_HTTP_PORT=3000
      - MCP_HTTP_HOST=0.0.0.0
    deploy:
      resources:
        limits: { memory: 256m }

  playwright:
    image: mcr.microsoft.com/playwright/mcp:latest
    container_name: mcp-playwright
    restart: unless-stopped
    init: true
    networks: [mcp]
    dns: [1.1.1.1, 9.9.9.9]
    ports:
      - "0.0.0.0:8931:8931"
    entrypoint: ["node", "/app/cli.js"]
    command:
      - --headless
      - --browser=chromium
      - --no-sandbox
      - --isolated
      - --port=8931
      - --host=0.0.0.0
      - --allowed-hosts=pw-xxxxxxxx.yamyam.com   # CHANGE THIS
    shm_size: 1g
    deploy:
      resources:
        limits: { memory: 2g }

  crawl4ai:
    image: unclecode/crawl4ai:0.9.4
    container_name: mcp-crawl4ai
    restart: unless-stopped
    networks: [mcp]
    dns: [1.1.1.1, 9.9.9.9]
    environment:
      - CRAWL4AI_API_TOKEN=MCP_KEY             # CHANGE THIS
    shm_size: 1g
    deploy:
      resources:
        limits: { memory: 3g }

  crawl4ai-bridge:
    image: ghcr.io/sparfenyuk/mcp-proxy:v0.12.0
    container_name: mcp-crawl4ai-bridge
    restart: unless-stopped
    networks: [mcp]
    dns: [1.1.1.1, 9.9.9.9]
    depends_on: [crawl4ai]
    ports:
      - "0.0.0.0:11241:8096"
    command:
      - --host=0.0.0.0
      - --port=8096
      - --
      - mcp-proxy
      - --headers
      - Authorization
      - Bearer MCP_KEY                         # CHANGE THIS
      - http://crawl4ai:11235/mcp/sse
    deploy:
      resources:
        limits: { memory: 256m }

networks:
  mcp:
    driver: bridge
    ipam:
      config:
        - subnet: 10.251.0.0/24

volumes:
  tube-cache:
```

#### The crawl4ai
Few notes for crawl4ai as it's a little more pain in the ass..


**1. It binds only to loopback**
Since 0.9.x, if you don't set `CRAWL4AI_API_TOKEN`, it listens only on `127.0.0.1` inside the container. Container shows healthy, port is published, and nothing works. The logs say it clearly though
```bash
entrypoint: no CRAWL4AI_API_TOKEN set; binding loopback only (127.0.0.1:11235).
entrypoint: WARNING: this is the CONTAINER's loopback - published ports will NOT work
```
The fix is actually a security bonus. Set the token to the same `MCP_KEY` you use for Cloudflare, and crawl4ai validates the key on its own too.


**2. It speaks only SSE**
crawl4ai exposes MCP over the older SSE transport (`/mcp/sse`). claude.ai refused to connect to it, even though curl worked just fine through Cloudflare. So we put [mcp-proxy](https://github.com/sparfenyuk/mcp-proxy) in front of it. It connects to crawl4ai over SSE (with the key) and exposes it as Streamable HTTP on `/mcp`. That's why crawl4ai itself doesn't publish any port.


### Test from the cloudflared LXC
This is the only machine that should be able to reach the tools
```bash
# searxng
curl -s --max-time 5 http://10.1.1.23:3010/health; echo

# youtube
curl -s --max-time 5 -X POST http://10.1.1.23:8091/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1"}}}' | head -c 300; echo

# crawl4ai bridge
curl -s --max-time 10 -X POST http://10.1.1.23:11241/mcp \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1"}}}' | head -c 300; echo
```
You should get `serverInfo` back. Same for Playwright on 8931, just add `-H 'Host: pw-xxxxxxxx.yamyam.com'`.


Now try the same from your PC. It has to **time out**
```bash
curl -s --max-time 5 http://10.1.1.23:3010/health || echo "BLOCKED - good"
```


And check the outbound rules from inside a container
```bash
# internet - should be 200
docker exec mcp-searxng node -e "fetch('https://example.com').then(r=>console.log('internet:',r.status))"

# LAN - should be blocked
docker exec mcp-searxng node -e "fetch('http://10.1.1.23:3000',{signal:AbortSignal.timeout(5000)}).then(r=>console.log('LAN OPEN:',r.status)).catch(e=>console.log('LAN blocked - good'))"
```


## Cloudflare
### Tunnel routes
Zero Trust -> Networks -> Tunnels -> your tunnel -> Routes -> Add route -> Published application


| Subdomain | Path | Service URL |
|---|---|---|
| `yt-xxxxxxxx` | `^/mcp` | `http://10.1.1.23:8091` |
| `search-xxxxxxxx` | `^/mcp` | `http://10.1.1.23:3010` |
| `pw-xxxxxxxx` | `^/mcp` | `http://10.1.1.23:8931` |
| `c4ai-xxxxxxxx` | `^/mcp` | `http://10.1.1.23:11241` |


The `Path` field makes sure only the MCP endpoint goes through, the crawl4ai REST API and playground stay hidden.


If you get `502` from Cloudflare, your route points to a wrong port. Ask me how I know.


### WAF rule
Your domain -> Security -> Security rules -> Create rule -> Custom rules
```
(http.host in {"yt-xxxxxxxx.yamyam.com" "search-xxxxxxxx.yamyam.com" "pw-xxxxxxxx.yamyam.com" "c4ai-xxxxxxxx.yamyam.com"})
and not (
  any(http.request.headers["authorization"][*] eq "Bearer MCP_KEY")
  and (ip.src in {160.79.104.0/21} or ip.src.country eq "SK")
)
```
Action: **Block**


In human language: if somebody wants to talk to one of the MCP hostnames, they need the key AND they need to come from Anthropic's IP range (or from Slovakia, so I can use it from home even when my ISP changes my IP).


Double check the Anthropic range in their docs, it can change.


Why the key at all if we already filter Anthropic IPs? Because **every Claude user** comes from that range. Random subdomains make it harder to find, but that's not a password. The key is.


Test it from your PC
```bash
URL=https://search-xxxxxxxx.yamyam.com/mcp
BODY='{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1"}}}'

# no key - 403
curl -s -o /dev/null -w '%{http_code}\n' -X POST $URL \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -d "$BODY"

# with key - 200
read -rs -p "Key: " KEY; echo
curl -s -o /dev/null -w '%{http_code}\n' -X POST $URL -H "Authorization: Bearer $KEY" \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' -d "$BODY"
unset KEY
```
Also try the URL from your phone on mobile data, you should see Cloudflare's "Sorry, you have been blocked" page.


## Claude connectors
Settings -> Connectors -> Add -> Custom connector


| Name | URL |
|---|---|
| YouTube | `https://yt-xxxxxxxx.yamyam.com/mcp` |
| Search | `https://search-xxxxxxxx.yamyam.com/mcp` |
| Browser | `https://pw-xxxxxxxx.yamyam.com/mcp` |
| crawl4ai | `https://c4ai-xxxxxxxx.yamyam.com/mcp` |


* Authentication: **No sign-in**
* Request headers -> Add header -> pick **authorization** from the dropdown
* Value: `Bearer MCP_KEY` - yes, including the word `Bearer` and the space, the value is sent exactly as entered


I originally wanted a custom header like `x-mcp-key`, but custom header names need approval from Anthropic. `authorization` is in the dropdown and works right away.


Last thing, set tool permissions. YouTube and search can run on their own, they only read. **Browser should always ask for approval**, it can click and fill forms.


## Check out other posts in the series
{% include post-series.html %}
