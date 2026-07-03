# Cloud Firewall — `sweety-firewall`

DigitalOcean **Cloud Firewall** applied to the **backend droplets only**. From a DevOps perspective, this is the **defense-in-depth** layer that ensures backend servers accept traffic only from inside the VPC — not directly from the public internet.

See also: [VPC](vpc.md) · [DigitalOcean](digitalocean.md) · [Load Balancer](loadbalancer.md) · [Tailscale](tailscale.md) · [Setup Guide](readme.md)

---

## DevOps overview

In a production-style setup, you never expose every server to the internet. You expose **one entry point** (the load balancer) and **lock down everything behind it**.

```
                    PUBLIC INTERNET
                          │
                          │  HTTP :80 allowed
                          ▼
              ┌───────────────────────┐
              │  ubuntu-s-loadbalance │  ◄── NO firewall (or separate rules)
              │  142.93.192.78      │      Public-facing entry point
              └───────────┬───────────┘
                          │
                          │  private VPC traffic only
                          │  10.116.0.0/20
                          ▼
    ┌─────────────────────────────────────────────────┐
    │         sweety-firewall (5 rules)               │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
    │  │  DROP1   │  │  drop2   │  │  drop3   │      │
    │  │ .0.2     │  │ .0.4     │  │ .0.3     │      │
    │  └──────────┘  └──────────┘  └──────────┘      │
    │  Backends — HTTP blocked from public internet   │
    └─────────────────────────────────────────────────┘
```

| Principle | How this project applies it |
|-----------|----------------------------|
| **Least privilege** | Backends only accept TCP from the VPC subnet — not from the world |
| **Single entry point** | Users hit the load balancer public IP; backends stay private |
| **Centralized policy** | One firewall (`sweety-firewall`) applied to all 3 backends consistently |
| **Separation of roles** | LB is public; backends are protected |

---

## Firewall summary

| Setting | Value |
|---------|-------|
| **Name** | `sweety-firewall` |
| **Total rules** | 5 (2 inbound + 3 outbound) |
| **Protected droplets** | 3 backends (not the load balancer) |

---

## Inbound rules

![sweety-firewall inbound and outbound rules](firewall-rules.png)

Inbound rules control **what traffic is allowed to reach** the backend droplets. Everything not listed is **denied by default**.

### Rule 1 — All TCP from VPC subnet

| Field | Value |
|-------|-------|
| Type | All TCP |
| Protocol | TCP |
| Port range | All ports |
| Sources | `10.116.0.0/20` |

**DevOps meaning:**

This is the **core rule** for this architecture. It allows any droplet inside `sweety-vpc` (including the load balancer at `10.116.0.5`) to reach the backends over **any TCP port**.

Why it matters:

- The load balancer can `reverse_proxy` to backends on port **80**
- Health checks (`health_uri /`) from Caddy work over the private network
- Traffic stays inside the VPC — aligned with [vpc.md](vpc.md)
- **Public HTTP to backend public IPs is blocked** — a user cannot bypass the load balancer by hitting `161.35.13.56` directly

```
✅  loadbalance (10.116.0.5) → drop1 (10.116.0.2:80)   ALLOWED
✅  loadbalance (10.116.0.5) → drop2 (10.116.0.4:80)   ALLOWED
❌  random user → drop1 (161.35.13.56:80)              BLOCKED
```

### Rule 2 — SSH from anywhere

| Field | Value |
|-------|-------|
| Type | SSH |
| Protocol | TCP |
| Port range | 22 |
| Sources | All IPv4, All IPv6 |

**DevOps meaning:**

Allows **remote administration** via Termius/SSH from any IP. Needed so you can manage backends without being on the VPC.

**Production note:** In a real environment, restrict SSH to your office IP or a bastion host instead of `All IPv4 / All IPv6`. For a lab project, open SSH is common for convenience.

---

## Outbound rules

Outbound rules control **what traffic can leave** the backend droplets.

| Rule | Protocol | Port range | Destinations |
|------|----------|------------|--------------|
| 1 | ICMP | — | All IPv4, All IPv6 |
| 2 | All TCP | All ports | All IPv4, All IPv6 |
| 3 | All UDP | All ports | All IPv4, All IPv6 |

**DevOps meaning:**

Outbound is **permissive** — backends can:

- Run `apt update` / `apt upgrade` (TCP 443 to package repos)
- Resolve DNS (UDP 53)
- Respond to ping / diagnostics (ICMP)
- Pull Caddy GPG keys and packages during install

This follows the common pattern: **restrict inbound tightly, allow outbound for operations**.

---

## Droplets protected by this firewall

![sweety-firewall applied to 3 backend droplets](firewall-droplets.png)

The firewall is attached to **backend servers only** — not the load balancer:

| Droplet | Public IP | Region | Size | Role |
|---------|-----------|--------|------|------|
| `ubuntu-s-DROP1` | `161.35.13.56` | NYC1 | 512 MB / 10 GB | Backend 1 |
| `ubuntu-s-drop2` | `142.93.254.161` | NYC1 | 512 MB / 10 GB | Backend 2 |
| `ubuntu-s-drop3` | `174.138.32.157` | NYC1 | 512 MB / 10 GB | Backend 3 |

| Droplet | On firewall? | Why |
|---------|--------------|-----|
| `ubuntu-s-DROP1` | ✅ Yes | Backend — must be locked down |
| `ubuntu-s-drop2` | ✅ Yes | Backend — must be locked down |
| `ubuntu-s-drop3` | ✅ Yes | Backend — must be locked down |
| `ubuntu-s-loadbalance` | ❌ No | Public entry point — needs open port 80 from internet |

---

## How firewall + VPC + load balancer work together

These three pieces form a complete **secure traffic path**:

```
1. VPC (sweety-vpc)
   └── Gives each droplet a private IP (10.116.0.x)
   └── Enables internal routing between droplets

2. Load Balancer (ubuntu-s-loadbalance)
   └── Listens on public IP :80
   └── Forwards to backend private IPs in Caddyfile

3. Firewall (sweety-firewall)
   └── Blocks public HTTP to backends
   └── Allows only VPC-internal TCP to reach backends
```

**Without the firewall:** Backends would still have public IPs and port 80 open — anyone could bypass the load balancer.

**With the firewall:** Only the load balancer (inside `10.116.0.0/20`) can reach backend web ports. Defense in depth.

---

## Step 1 — Create the cloud firewall

1. DigitalOcean → **Networking** → **Firewalls** → **Create Firewall**
2. Name it **`sweety-firewall`**
3. Add **inbound rules** (see above)
4. Add **outbound rules** (see above)

---

## Step 2 — Attach backend droplets only

1. Open **`sweety-firewall`** → **Droplets** tab
2. Click **Add Droplets**
3. Select:
   - `ubuntu-s-DROP1`
   - `ubuntu-s-drop2`
   - `ubuntu-s-drop3`
4. Do **not** add `ubuntu-s-loadbalance`

---

## Step 3 — Verify the setup

### Test 1 — Backends blocked from public HTTP

From your local machine, try hitting a backend **public IP** directly:

```bash
curl http://161.35.13.56
curl http://142.93.254.161
curl http://174.138.32.157
```

Expected: **timeout or connection refused** — firewall blocks public HTTP.

### Test 2 — Load balancer still works

```bash
curl http://142.93.192.78
```

Expected: **HTML response** from one of the backends via the load balancer.

### Test 3 — Private path works (from load balancer)

SSH into the load balancer and curl private IPs:

```bash
curl http://10.116.0.2
curl http://10.116.0.3
curl http://10.116.0.4
```

Expected: **HTML from each backend** — VPC + firewall allow internal traffic.

### Test 4 — SSH still works

```bash
ssh root@161.35.13.56
```

Expected: **SSH connects** — port 22 is allowed from all IPs.

---

## DevOps best practices applied

| Practice | This project |
|----------|--------------|
| Default deny inbound | Only VPC TCP + SSH allowed |
| Network segmentation | VPC isolates internal traffic |
| Single public entry | Load balancer only |
| Consistent policy | One firewall on all backends |
| Operational outbound | apt, DNS, ICMP allowed for maintenance |

### Possible improvements for production

- Restrict SSH to specific admin IPs instead of `All IPv4 / All IPv6`
- Add a firewall on the load balancer allowing only port 80 + 443 from the internet
- Use a bastion host for SSH instead of open port 22 on backends
- Add monitoring/alerting when firewall rules change

---

## Quick reference

| Traffic path | Allowed? | Rule |
|--------------|----------|------|
| Internet → load balancer :80 | ✅ | LB has no restrictive firewall |
| Internet → backend :80 (public IP) | ❌ | No inbound rule for public HTTP |
| LB → backend :80 (private IP) | ✅ | Inbound TCP from `10.116.0.0/20` |
| Termius → backend :22 | ✅ | Inbound SSH from all IPs |
| Backend → internet (apt update) | ✅ | Outbound TCP/UDP/ICMP allowed |

---

## Checklist

- [ ] Firewall `sweety-firewall` created with 5 rules
- [ ] Inbound: All TCP from `10.116.0.0/20`
- [ ] Inbound: SSH (22) from all IPs
- [ ] Outbound: ICMP, TCP, UDP to all destinations
- [ ] Attached to `DROP1`, `drop2`, `drop3` — **not** loadbalance
- [ ] Public curl to backend IPs fails
- [ ] Curl to load balancer public IP works
- [ ] SSH to backends via Termius still works
