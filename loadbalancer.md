# Load Balancer Droplet Setup

Dedicated guide for the **loadbalance** droplet — a fresh VPS that runs **only Caddy** as a DIY reverse proxy with health checks. No static HTML on this server.

See also: [Backend setup](readme.md) · [DigitalOcean](digitalocean.md) · [VPC](vpc.md) · [Firewall](firewall.md) · [Tailscale](tailscale.md) · [Tools used](tools.md)

---

## Overview

| Item | Value |
|------|-------|
| Host name | `loadbalance` |
| Hostname on server | `ubuntu-s-loadbalance` |
| Public IP (eth0) | `142.93.192.78` |
| Private IP (eth1) | `10.116.0.5` |
| Config file | `/etc/caddy/Caddyfile` |
| Role | Reverse proxy only — no `file_server` |

Backend droplets use **private IPs** in the Caddyfile so traffic stays on DigitalOcean's internal network. This is safer than exposing backend public IPs to the load balancer.

---

## Step 1 — Create the load balancer droplet

1. In DigitalOcean, create a **new Ubuntu droplet** (same region as `drop1`, `drop2`, `drop3`).
2. Label it **loadbalance**.
3. Add it as a host in Termius (see [tools.md](tools.md)).
4. SSH in:

```bash
ssh root@142.93.192.78
```

---

## Step 2 — Update system and install Caddy

```bash
sudo apt update
sudo apt upgrade -y
```

Install Caddy using the official repository (same commands as backend droplets — see [readme.md Step 2.3](readme.md#23--install-caddy)):

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

---

## Step 3 — Check network interfaces (public vs private IP)

Before editing the Caddyfile, confirm which interface is public and which is private. On the load balancer droplet:

```bash
ifconfig
```

![ifconfig output on loadbalance — eth0 public IP and eth1 private IP](loadbalancer-ifconfig.png)

### What you should see

| Interface | IP | Purpose |
|-----------|-----|---------|
| **eth0** | `142.93.192.78` | Public IP — clients hit this from the internet |
| **eth1** | `10.116.0.5` | Private IP — internal VPC traffic between droplets |
| **lo** | `127.0.0.1` | Loopback — local only |

### Why private IPs for backends?

- Traffic between load balancer and backends stays **inside DigitalOcean's private network**
- Backend servers are **not exposed** directly to the public internet for LB traffic
- Lower latency and better **security** within the same region/VPC

Backend private IPs used in this setup (from VPC `sweety-vpc`):

| Droplet | Private IP |
|---------|------------|
| drop1 (ubuntu-s-DROP1) | `10.116.0.2` |
| drop3 (ubuntu-s-drop3) | `10.116.0.3` |
| drop2 (ubuntu-s-drop2) | `10.116.0.4` |

Full VPC details: [vpc.md](vpc.md)

> Get each backend's private IP with `ifconfig` on that droplet (look at **eth1**).

---

## Step 4 — Edit the Caddyfile

Open the Caddy config on the load balancer:

```bash
cd /etc/caddy
sudo nano Caddyfile
```

![Editing Caddyfile in nano on loadbalance via Termius](loadbalancer-caddyfile.png)

Paste this configuration (uses **private IPs**, not public):

```caddyfile
# The Caddyfile is an easy way to configure your Caddy web server.
#
# Unless the file starts with a global options block, the first
# uncommented line is always the address of your site.
#
# To use your own domain name (with automatic HTTPS), first make
# sure your domain's A/AAAA DNS records are properly pointed to
# this machine's public IP, then replace ":80" below with your
# domain name.

:80 {
        # Set this path to your site's directory.
        reverse_proxy 10.116.0.2:80 10.116.0.3:80 10.116.0.4:80 {

                lb_policy round_robin
                health_uri /
                health_interval 10s
        }

}

# Refer to the Caddy docs for more information:
# https://caddyserver.com/docs/caddyfile
```

Save and exit nano: `Ctrl+O` → Enter → `Ctrl+X`

### Config explained

| Directive | Meaning |
|-----------|---------|
| `:80` | Listen on port 80 (public traffic via eth0) |
| `reverse_proxy 10.116.0.2:80 ...` | Forward to three backends over **private network** |
| `lb_policy round_robin` | Rotate requests evenly across healthy backends |
| `health_uri /` | Health check hits `/` on each backend |
| `health_interval 10s` | Re-check backend health every 10 seconds |

---

## Step 5 — Restart Caddy

Apply the new config:

```bash
sudo systemctl restart caddy.service
```

Check that Caddy is running:

```bash
sudo systemctl status caddy
```

Optional — verify the service name and reload without full restart:

```bash
sudo systemctl reload caddy
```

---

## Step 6 — Verify load balancing

From your local machine, hit the **load balancer public IP**:

```bash
curl http://142.93.192.78
```

Run it several times or refresh in a browser. You should see responses from **drop1**, **drop2**, or **drop3** rotating (round robin).

```bash
# Run multiple times to see different backends
for i in {1..6}; do curl -s http://142.93.192.78 | grep -o '<h1>.*</h1>'; done
```

---

## Step 7 — Test health checks

Stop Caddy on one backend (e.g. drop2):

```bash
# On drop2 only
sudo systemctl stop caddy
```

Hit the load balancer again — traffic should only go to healthy backends:

```bash
curl http://142.93.192.78
```

Restore the backend:

```bash
sudo systemctl start caddy
```

---

## Quick reference

| Task | Command |
|------|---------|
| Check network | `ifconfig` |
| Edit config | `sudo nano /etc/caddy/Caddyfile` |
| Restart Caddy | `sudo systemctl restart caddy.service` |
| Check status | `sudo systemctl status caddy` |
| Test LB | `curl http://142.93.192.78` |

---

## Checklist

- [ ] `loadbalance` droplet created in DigitalOcean
- [ ] Caddy installed (no HTML / no `file_server` on this droplet)
- [ ] `ifconfig` confirms eth0 (public) and eth1 (private)
- [ ] Caddyfile uses backend **private IPs** (`10.116.0.2`–`.4`)
- [ ] `round_robin` + health checks configured
- [ ] `systemctl restart caddy.service` succeeded
- [ ] Browser/curl shows rotating backend responses
