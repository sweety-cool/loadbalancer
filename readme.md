# DIY Self-Hosted Load Balancer on DigitalOcean

A step-by-step guide to building a self-hosted load balancer with health checks using DigitalOcean droplets, Caddy static file servers, and a dedicated load balancer VPS.

**Tools used:** [Termius, DigitalOcean, Caddy →](tools.md)  
**DigitalOcean:** [Droplet setup & inventory →](digitalocean.md)  
**VPC:** [Private networking →](vpc.md)  
**Firewall:** [Cloud firewall (DevOps) →](firewall.md)  
**Tailscale:** [Mesh networking overlay →](tailscale.md)  
**Load balancer:** [Dedicated LB droplet setup →](loadbalancer.md)  
**DNS Load Balancing:** [DNS Redundancy & Load Balancing →](DNS-loadbalancing/readme.md)

---

## Architecture Overview

| Droplet | Role | Purpose |
|---------|------|---------|
| DROP1 | Backend | Caddy + static HTML (VPS 1) |
| VPS2  | Backend | Caddy + static HTML (VPS 2) |
| VPS3  | Backend | Caddy + static HTML (VPS 3) |
| LB    | Load Balancer | Routes traffic to healthy backend droplets |

```
                    ┌─────────────────┐
   Client ────────► │  Load Balancer  │
                    │     (LB VPS)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         ┌─────────┐   ┌─────────┐   ┌─────────┐
         │  DROP1  │   │  VPS2   │   │  VPS3   │
         │  Caddy  │   │  Caddy  │   │  Caddy  │
         │ HTML A  │   │ HTML B  │   │ HTML C  │
         └─────────┘   └─────────┘   └─────────┘
```

---

## Step 1 — Create the Droplets

Create the following droplets in DigitalOcean:

1. **DROP1** (`ubuntu-s-DROP1`) — Backend server 1
2. **drop2** (`ubuntu-s-drop2`) — Backend server 2
3. **drop3** (`ubuntu-s-drop3`) — Backend server 3
4. **loadbalance** (`ubuntu-s-loadbalance`) — Dedicated load balancer VPS

Full DigitalOcean guide with droplet IPs, dashboard screenshot, and checklist:

**→ [digitalocean.md](digitalocean.md)**

---

## Step 2 — Backend Droplets (DROP1, VPS2, VPS3)

Repeat the steps below on **each backend droplet**. Only the HTML content changes per VPS.

### 2.1 — SSH into the droplet

```bash
ssh root@<DROPLET_IP>
```

### 2.2 — Update the system

```bash
sudo apt update
sudo apt upgrade -y
```

### 2.3 — Install Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list

sudo apt update
sudo apt install -y caddy
```

### 2.4 — Create the HTML page

Create the web root directory and `index.html`. Change the heading text for each VPS.

**DROP1** — `/var/www/html/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<body>
  <div class="container">
    <h1>MY VPS DROP1</h1>
    <p>This is a simple example of a VPS1 file.</p>
  </div>
</body>
</html>
```

**VPS2** — `/var/www/html/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<body>
  <div class="container">
    <h1>MY VPS VPS2</h1>
    <p>This is a simple example of a VPS2 file.</p>
  </div>
</body>
</html>
```

**VPS3** — `/var/www/html/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<body>
  <div class="container">
    <h1>MY VPS VPS3</h1>
    <p>This is a simple example of a VPS3 file.</p>
  </div>
</body>
</html>
```

Create the file:

```bash
sudo mkdir -p /var/www/html
sudo nano /var/www/html/index.html
```

### 2.5 — Configure Caddyfile

Edit `/etc/caddy/Caddyfile` — **same config on all backend droplets**:

```caddyfile
:80 {
        root * /var/www/html
        file_server
}
```

Apply the config:

```bash
sudo systemctl reload caddy
sudo systemctl status caddy
```

### 2.6 — Verify the backend

Open in a browser or curl from your machine:

```bash
curl http://<DROPLET_IP>
```

You should see the VPS-specific `<h1>` heading.

---

## Step 3 — Load Balancer Droplet (LB VPS)

Create a **fresh VPS** in DigitalOcean named `loadbalance`. This droplet runs **only Caddy** as a reverse proxy — no static HTML.

Full step-by-step guide with Caddyfile, private IP setup, and screenshots:

**→ [loadbalancer.md](loadbalancer.md)**

Summary:
- Use **private IPs** (`10.116.0.x` on eth1) for backend targets — not public IPs
- `lb_policy round_robin` with `health_uri /` every `10s`
- Apply config: `sudo systemctl restart caddy.service`
- Test via load balancer public IP: `curl http://142.93.192.78`

---

## Step 4 — DNS Load Balancing & Redundancy

Introduce a second load balancer for high availability (HA). Configure DNS Round-Robin A records to point `lb.klyrabit.com` to both load balancers so that if one goes down, the other continues serving requests.

Full step-by-step guide with Caddy configuration, DNS records, and failover verification:

**→ [DNS-loadbalancing/readme.md](DNS-loadbalancing/readme.md)**

---

## Quick Reference — Commands per Backend Droplet

| Step | Command |
|------|---------|
| Update system | `sudo apt update && sudo apt upgrade -y` |
| Install Caddy | See Step 2.3 |
| HTML file | `/var/www/html/index.html` |
| Caddy config | `/etc/caddy/Caddyfile` |
| Reload Caddy | `sudo systemctl reload caddy` |
| Check status | `sudo systemctl status caddy` |

---

## Droplet Checklist

- [ ] DROP1 created — Caddy + HTML A
- [ ] VPS2 created — Caddy + HTML B
- [ ] VPS3 created — Caddy + HTML C
- [ ] LB droplet created — reverse proxy only
- [ ] All backend droplets respond on port 80
- [ ] Load balancer distributes traffic across backends
- [ ] Health check removes unhealthy backends
- [ ] DNS Load Balancing: Second LB (`lb-server-2`) created and configured
- [ ] DNS Load Balancing: Round Robin DNS A records pointed to both LBs
- [ ] DNS Load Balancing: Failover verification completed successfully

---

## Notes

- All **backend** droplets use the **same Caddyfile**; only `index.html` differs.
- The **LB droplet** uses Caddy as a **reverse proxy** with active health checks — no static files. See [loadbalancer.md](loadbalancer.md).
- Backend targets use **private IPs** on eth1 for security (traffic stays on DO internal network).
- Keep all droplets in the **same DigitalOcean region** for best performance.
- Open **port 80** in each droplet's firewall / DigitalOcean cloud firewall if traffic does not reach the servers.
