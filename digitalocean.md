# DigitalOcean — Droplet Setup

All VPS for this project are hosted on [DigitalOcean](https://www.digitalocean.com). Four droplets run in the same region: one load balancer and three backend servers.

See also: [Setup Guide](readme.md) · [VPC](vpc.md) · [Firewall](firewall.md) · [Tailscale](tailscale.md) · [Load Balancer](loadbalancer.md) · [Tools](tools.md)

---

## Droplets dashboard

After creating all servers, the DigitalOcean **Droplets** page shows every machine in one place:

![DigitalOcean Droplets dashboard — all four VPS listed](digitalocean-droplets.png)

---

## Droplet inventory

| Name | Role | Public IP | Region | Image | Size | Created |
|------|------|-----------|--------|-------|------|---------|
| `ubuntu-s-loadbalance` | Load balancer | `142.93.192.78` | NYC1 | Ubuntu 24.04 (LTS) x64 | 512 MB / 10 GB | 6 hours ago |
| `ubuntu-s-drop3` | Backend VPS 3 | `174.138.32.157` | NYC1 | Ubuntu 24.04 (LTS) x64 | 512 MB / 10 GB | 10 hours ago |
| `ubuntu-s-drop2` | Backend VPS 2 | `142.93.254.161` | NYC1 | Ubuntu 24.04 (LTS) x64 | 512 MB / 10 GB | 10 hours ago |
| `ubuntu-s-DROP1` | Backend VPS 1 | `161.35.13.56` | NYC1 | Ubuntu 24.04 (LTS) x64 | 512 MB / 10 GB | 10 hours ago |

All droplets show **Active** status (green dot) in the dashboard.

---

## Architecture on DigitalOcean

```
Internet
    │
    ▼
142.93.192.78  ──►  ubuntu-s-loadbalance  (reverse proxy)
                           │
              ┌────────────┼────────────┐
              │  private network      │
              ▼            ▼            ▼
        10.116.0.2   10.116.0.3   10.116.0.4
        ubuntu-s-    ubuntu-s-    ubuntu-s-
        DROP1        drop3        drop2
        161.35.13.56 142.93.254.161 174.138.32.157
        (Caddy+HTML) (Caddy+HTML)  (Caddy+HTML)
```

- **Public IPs** — used for SSH (Termius) and for clients hitting the load balancer
- **Private IPs** — used inside the Caddyfile on `loadbalance` for backend traffic (see [loadbalancer.md](loadbalancer.md))

---

## Step 1 — Create backend droplets (drop1, drop2, drop3)

1. Log in to [DigitalOcean Cloud](https://cloud.digitalocean.com).
2. Go to **Droplets** → **Create Droplet**.
3. Choose:
   - **Region:** NYC1 (same region for all droplets)
   - **Image:** Ubuntu 24.04 (LTS) x64
   - **Size:** Basic — 512 MB RAM / 10 GB disk ($4/mo)
   - **Authentication:** SSH key (recommended) or password
4. Set the **hostname** (e.g. `ubuntu-s-DROP1`).
5. Click **Create Droplet**.
6. Repeat for `ubuntu-s-drop2` and `ubuntu-s-drop3`.

After each droplet is created, note its **public IP** from the dashboard.

---

## Step 2 — Create the load balancer droplet

1. **Droplets** → **Create Droplet** again.
2. Use the **same region** (NYC1) and **same image/size** as the backends.
3. Set hostname to `ubuntu-s-loadbalance`.
4. Create the droplet and note the public IP: `142.93.192.78`.

This droplet runs **only Caddy as a reverse proxy** — no static HTML. See [loadbalancer.md](loadbalancer.md).

---

## Step 3 — Add droplets to Termius

Copy each **public IP** from the DigitalOcean dashboard into Termius:

| Termius label | DigitalOcean name | Public IP |
|---------------|-------------------|-----------|
| `loadbalance` | ubuntu-s-loadbalance | `142.93.192.78` |
| `drop1` | ubuntu-s-DROP1 | `161.35.13.56` |
| `drop2` | ubuntu-s-drop2 | `142.93.254.161` |
| `drop3` | ubuntu-s-drop3 | `174.138.32.157` |

See [tools.md](tools.md) for Termius setup.

---

## Step 4 — Get private IPs

On each droplet, run:

```bash
ifconfig
```

Look at **eth1** for the private IP. These are used in the load balancer Caddyfile:

| Droplet | Private IP (eth1) |
|---------|-------------------|
| ubuntu-s-DROP1 | `10.116.0.2` |
| ubuntu-s-drop3 | `10.116.0.3` |
| ubuntu-s-drop2 | `10.116.0.4` |
| ubuntu-s-loadbalance | `10.116.0.5` |

See [vpc.md](vpc.md) for the full VPC overview and private networking explanation.

---

## Recommended settings

| Setting | Value | Why |
|---------|-------|-----|
| Region | NYC1 | All droplets in one region = low latency + private networking |
| OS | Ubuntu 24.04 LTS | Stable, well supported for Caddy |
| Size | 512 MB / 10 GB | Enough for static HTML + reverse proxy lab |
| Firewall | Allow port 80, 22 | HTTP for Caddy, SSH for Termius |

---

## Checklist

- [ ] `ubuntu-s-DROP1` created — public IP `161.35.13.56`
- [ ] `ubuntu-s-drop2` created — public IP `142.93.254.161`
- [ ] `ubuntu-s-drop3` created — public IP `174.138.32.157`
- [ ] `ubuntu-s-loadbalance` created — public IP `142.93.192.78`
- [ ] All droplets in **NYC1**, **Active** in dashboard
- [ ] Public IPs added to Termius
- [ ] Private IPs collected for load balancer Caddyfile
