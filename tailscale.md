# Tailscale — Mesh Networking Overlay

This project uses **Tailscale** alongside DigitalOcean **VPC** to build a secure mesh network. Your Mac and all backend droplets join the same tailnet, so you can SSH and manage servers over **private Tailscale IPs** — even when outbound firewalls or NAT would normally block direct connections.

See also: [VPC](vpc.md) · [Firewall](firewall.md) · [DigitalOcean](digitalocean.md) · [Tools](tools.md) · [Setup Guide](readme.md)

**Reference:** [How Tailscale works](https://tailscale.com/blog/how-tailscale-works) (Tailscale blog)

---

## DevOps overview

In this project, two networking layers work together:

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1 — VPC (sweety-vpc)          Traditional / datacenter   │
│  10.116.0.0/20 — droplets talk inside DigitalOcean              │
│  Used for: load balancer → backend traffic (production path)    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2 — Tailscale (tailnet)       Mesh / overlay network     │
│  100.x.x.x — encrypted WireGuard tunnels between all nodes      │
│  Used for: admin access, Mac → droplet SSH, node discovery       │
└─────────────────────────────────────────────────────────────────┘
```

| Layer | Type | Scope | Primary use in this project |
|-------|------|-------|----------------------------|
| **VPC** | Traditional network | Same cloud region | LB → backend HTTP over private IPs |
| **Tailscale** | Mesh overlay | Anywhere (Mac + cloud) | Secure admin access, nodes know each other |

---

## How Tailscale works (summary)

Based on [How Tailscale works](https://tailscale.com/blog/how-tailscale-works):

### Data plane — WireGuard mesh

- Tailscale uses **WireGuard** to create lightweight encrypted tunnels between nodes.
- Unlike traditional **hub-and-spoke VPN** (all traffic through one concentrator), Tailscale builds a **point-to-point mesh** — nodes talk directly when possible.
- Result: **lower latency**, no central bottleneck for data traffic.

### Control plane — coordination server

- Each node generates a **public/private keypair**. Private keys **never leave the node**.
- Nodes register with Tailscale's **coordination server** (`login.tailscale.com`) — a key exchange service, not a traffic funnel.
- You log in via OAuth (Gmail, etc.). Tailscale downloads peer public keys to each node.
- **Control plane is centralized; data plane is distributed mesh.**

### NAT traversal & DERP relays

- Real-world nodes sit behind NAT/firewalls with no open inbound ports.
- Tailscale uses **STUN/ICE** to punch through NAT for direct peer connections.
- If UDP is blocked, **DERP relays** forward already-encrypted traffic (Tailscale cannot decrypt it).

### ACLs & zero trust

- Security policy lives on the coordination server and is enforced **on each node**.
- Unauthorized machines cannot even initiate a WireGuard handshake without the right public key.
- Mesh networking + per-node ACLs = **zero trust** model.

> Full technical deep-dive: [tailscale.com/blog/how-tailscale-works](https://tailscale.com/blog/how-tailscale-works)

---

## Our Tailscale setup

**Account:** `sweety127ahmed@gmail.com`  
**Plan:** Free  
**Tailscale version:** `1.98.8` on all nodes

### Mac app — connected devices

Tailscale runs on your Mac. All mesh nodes appear in the **Devices** panel with green online status:

![Tailscale Mac app — mesh devices list](tailscale-mac-devices.png)

| Device | Tailscale IP | OS | Role |
|--------|--------------|-----|------|
| `sweety-mac` | `100.124.44.3` | macOS 26.1.0 | Local admin machine |
| `ubuntu-s-drop1` | `100.80.105.50` | Linux 6.8.0-124 | Backend VPS 1 |
| `ubuntu-s-drop2` | `100.107.34.9` | Linux 6.8.0-134 | Backend VPS 2 |
| `ubuntu-s-drop3` | `100.122.84.2` | Linux 6.8.0-124 | Backend VPS 3 |

From your Mac you can SSH to any droplet using Tailscale IP — **no public IP required for admin**:

```bash
ssh root@100.80.105.50    # drop1 via Tailscale
ssh root@100.107.34.9     # drop2 via Tailscale
ssh root@100.122.84.2     # drop3 via Tailscale
```

> **Note:** `ubuntu-s-loadbalance` is **not** on Tailscale in this setup — only backends + Mac. The load balancer is managed via public IP or Termius.

---

## Admin console — browser UI

Manage the tailnet from [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines):

![Tailscale admin console in browser — Machines tab](tailscale-browser-ui.png)

The **Machines** tab shows all 4 connected nodes, Tailscale IPs, OS versions, and connection status. Use this for:

- Adding/removing devices
- **Access controls (ACLs)**
- DNS settings (MagicDNS)
- Audit logs
- Disabling key expiry on servers

---

## Machine details — `ubuntu-s-drop1` example

Click any machine in the admin console to see full network details:

![Tailscale machine details — ubuntu-s-drop1](tailscale-drop1-details.png)

| Field | Value | DevOps meaning |
|-------|-------|----------------|
| **Tailscale IPv4** | `100.80.105.50` | Stable private admin IP — use this from Mac |
| **Tailscale IPv6** | `fd7a:115c:a1e0::1b32:694f` | IPv6 on the mesh |
| **MagicDNS** | `ubuntu-s-drop1.taila38709.ts.net` | Hostname resolves inside tailnet |
| **Endpoints** | `10.116.0.2:41641` (VPC), `161.35.13.56:41641` (public) | How Tailscale reaches this node |
| **Key expiry** | Disabled | Server stays connected without re-auth |
| **Exit node** | Not allowed | This node does not route internet for others |
| **Subnet routes** | None | Not bridging entire VPC into tailnet |

The **endpoints** list is important: Tailscale discovers **both** the VPC private IP (`10.116.0.2`) and public IP (`161.35.13.56`). It picks the best path — often the VPC route when both peers are in the same datacenter.

---

## Tailscale vs VPC — DevOps comparison

| | **VPC** (`sweety-vpc`) | **Tailscale** (mesh) |
|---|------------------------|----------------------|
| **Network type** | Traditional L3 private network | Overlay mesh (WireGuard) |
| **Scope** | Single cloud provider / region | Cross-region, cross-cloud, Mac ↔ cloud |
| **IP range** | `10.116.0.0/20` (DO-assigned) | `100.64.0.0/10` (Tailscale CGNAT) |
| **Setup** | Automatic when droplets share region | Install agent + login on each node |
| **Traffic path** | Datacenter internal routing | Encrypted peer-to-peer (or DERP relay) |
| **Firewall model** | Inbound rules on cloud firewall | Outbound-friendly — nodes **find each other** without opening inbound ports |
| **Best for** | Service-to-service inside cloud | Admin access, hybrid teams, remote ops |
| **Depends on** | Same cloud / same VPC | Internet + Tailscale coordination server |

### Key DevOps insight

- **VPC** = production traffic path (load balancer → backends). Fixed, fast, cloud-native.
- **Tailscale** = **management plane overlay**. Your Mac joins the mesh; droplets register; everyone knows each other without punching holes in **inbound** firewalls.

Tailscale is especially useful when **outbound** connectivity exists but **inbound** is blocked — nodes initiate connections outward to coordinate, then establish direct encrypted tunnels.

---

## When to use VPC vs Tailscale

### Use VPC when (3 scenarios — highest priority)

1. **Service-to-service traffic inside the same cloud** — load balancer forwarding to backends on `10.116.0.x`
2. **Lowest latency production path** — no overlay overhead; datacenter internal routing
3. **Cloud firewall enforcement** — `sweety-firewall` rules apply to VPC traffic (see [firewall.md](firewall.md))

### Use Tailscale when (2 scenarios — admin & hybrid)

1. **Admin / SSH from your Mac** — private access without exposing SSH to the world on Tailscale IPs
2. **Nodes in different networks need to know each other** — Mac at home, droplets in NYC1, no site-to-site VPN setup

### Use both together when (1 scenario — this project)

1. **Production on VPC + management on Tailscale** — the recommended hybrid:
   - HTTP traffic: `loadbalance` → backends via VPC private IPs
   - SSH/debug: `sweety-mac` → backends via Tailscale IPs
   - Cloud firewall blocks public HTTP to backends; Tailscale handles secure admin overlay

```
Production path:   User → LB (public) → backend (VPC private IP)
Admin path:        Mac (Tailscale) → backend (100.x.x.x)
```

---

## Tailscale & outbound firewall — why nodes "just know each other"

Traditional networking requires **inbound** firewall rules: "allow SSH from IP X to port 22."

Tailscale flips this for admin:

1. Each node runs the Tailscale agent and makes **outbound** connections to the coordination server.
2. Nodes exchange keys and endpoint info through the control plane.
3. Peer connections are established using **NAT traversal** — both sides connect outward; no inbound port open required on your Mac or droplet.
4. Once connected, traffic is **end-to-end encrypted** WireGuard.

From a DevOps perspective:

- You do **not** need to open inbound ports for admin mesh connectivity.
- Nodes in the same tailnet **discover each other** by identity (machine name + key), not by memorizing public IPs.
- This is ideal when [sweety-firewall](firewall.md) blocks most inbound traffic on backends — Tailscale still lets your Mac reach them over the overlay.

---

## How this project uses both layers

```
                         INTERNET
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
     Public HTTP      Tailscale         SSH (public)
     :80 to LB        mesh overlay      :22 (firewall rule)
            │               │
            ▼               ▼
   ubuntu-s-loadbalance   sweety-mac
   142.93.192.78          100.124.44.3
            │               │
            │ VPC           │ Tailscale (WireGuard)
            │ 10.116.0.x    │ 100.x.x.x
            ▼               ▼
   ┌────────────────────────────────────┐
   │  drop1        drop2        drop3   │
   │  10.116.0.2   10.116.0.4   10.116.0.3  ← VPC (production)
   │  100.80.x.x   100.107.x.x  100.122.x.x ← Tailscale (admin)
   └────────────────────────────────────┘
            ▲
            │ sweety-firewall (inbound VPC TCP only)
```

| Action | Use |
|--------|-----|
| User visits website | Public IP → load balancer → VPC private IP → backend |
| Mac SSH to drop1 | `ssh root@100.80.105.50` via Tailscale |
| Mac SSH to drop1 (alternative) | `ssh root@161.35.13.56` via public IP + Termius |
| LB health check to backend | VPC private IP (`10.116.0.x`) |
| Block public HTTP to backend | Cloud firewall — only VPC subnet allowed |

---

## Step 1 — Install Tailscale on each node

### On Ubuntu droplets (drop1, drop2, drop3)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the login URL to authenticate with `sweety127ahmed@gmail.com`.

### On Mac

1. Install Tailscale from [tailscale.com/download](https://tailscale.com/download) or Mac App Store.
2. Log in with the same account.
3. Toggle **Connected** in the menu bar.

---

## Step 2 — Disable key expiry on servers (recommended)

Servers should stay connected without manual re-auth:

1. Open [admin console → Machines](https://login.tailscale.com/admin/machines)
2. Click each Ubuntu droplet → **Machine settings**
3. Disable key expiry

All three backends show **Expiry disabled** in the admin UI.

---

## Step 3 — Verify mesh connectivity from Mac

```bash
# Ping over Tailscale
ping 100.80.105.50

# SSH over Tailscale (private — no public IP needed)
ssh root@100.80.105.50

# Or use MagicDNS hostname
ssh root@ubuntu-s-drop1.taila38709.ts.net
```

---

## Step 4 — Optional: restrict Tailscale ACLs

In the admin console → **Access controls**, define who can reach what:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["sweety-mac"],
      "dst": ["ubuntu-s-drop1:22", "ubuntu-s-drop2:22", "ubuntu-s-drop3:22"]
    }
  ]
}
```

This limits Tailscale to **SSH only** from your Mac — production HTTP still flows over VPC.

---

## Decision matrix — DevOps scenarios

| Scenario | VPC | Tailscale | Both |
|----------|-----|-----------|------|
| LB → backend HTTP in same region | ✅ | ❌ | — |
| Mac SSH to cloud droplets | ❌ | ✅ | — |
| Backend blocked from public internet | ✅ | — | ✅ |
| Multi-cloud (AWS + DO) private link | ❌ | ✅ | — |
| Compliance / audit central ACLs | ❌ | ✅ | — |
| Lowest latency service mesh | ✅ | ❌ | — |
| Remote team admin access | ❌ | ✅ | — |
| This load balancer lab | ✅ prod | ✅ admin | ✅ |

---

## 3-2-1 rule applied to networking

Adapt the backup **3-2-1** principle to network access:

| Rule | Application |
|------|-------------|
| **3** paths to think about | Public IP, VPC private IP, Tailscale IP |
| **2** network layers | VPC (production) + Tailscale (admin overlay) |
| **1** public entry point | Load balancer only — backends never exposed for HTTP |

Never rely on a single access method for production admin. Keep Termius (public SSH) **and** Tailscale (private mesh) as redundant admin paths.

---

## Checklist

- [ ] Tailscale installed on Mac — connected as `sweety-mac` (`100.124.44.3`)
- [ ] Tailscale installed on `drop1`, `drop2`, `drop3`
- [ ] All 4 machines show **Connected** in admin console
- [ ] Key expiry disabled on server nodes
- [ ] SSH from Mac via Tailscale IP works
- [ ] Production HTTP still uses VPC path (LB → `10.116.0.x`)
- [ ] Understand: VPC = production, Tailscale = admin mesh

---

## Further reading

- [How Tailscale works](https://tailscale.com/blog/how-tailscale-works) — WireGuard mesh, control plane, NAT traversal, DERP, ACLs
- [Tailscale docs](https://tailscale.com/kb/)
- [VPC in this project](vpc.md)
- [Cloud firewall in this project](firewall.md)
