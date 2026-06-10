# Pi Homelab — Architecture Plan

## Hardware Inventory

| Node | Hardware | RAM | Storage | Role | Status |
|------|----------|-----|---------|------|--------|
| `pi-fw` | Pi 5 + dual 2.5GbE HAT | 8GB | SD (OS only) | Router, firewall, Tailscale subnet router, Cloudflare Tunnel | Pending HAT |
| `pi-01` | Pi 5 | 8GB | 1TB NVMe (OS + data) | Comms: Mattermost, Stalwart Mail, HomelabBot | Up — 172.16.30.30 |
| `pi-nas` | Pi 5 + Radxa Penta SATA HAT | 8GB | 4× 512GB SATA SSD | Storage + services (TBD) | Up — 172.16.30.31, HAT bring-up in progress |
| `pi-02` | Pi 5 | 8GB | 512GB NVMe (OS + data) | Data: Nextcloud, Miniflux, Syncthing | Up — 172.16.30.32 |
| `pi-03` | Pi 4 | 8GB | TBD | Utility: Kavita, Uptime Kuma, Homepage | Not yet deployed |
| `pi-04` | Pi 4 | 8GB | TBD | Ansible control node, hot spare | Not yet deployed |
| MikroTik CRS310-8G+2S+IN | — | — | — | VLAN enforcement, 1GbE/10G fabric | In rack |

All Pi nodes run **Raspberry Pi OS 64-bit (Debian Bookworm)**.

Container runtime: **Docker Compose** initially; migrate to **Podman quadlets** once the stack is stable. `pi-fw` runs no containers — all services there are native systemd units.

**Storage notes:**
- `pi-01` and `pi-02` boot from NVMe (no SD cards).
- `pi-nas` uses the Radxa Penta SATA HAT (JMS585 PCIe bridge) — drives present as `/dev/sda`–`/dev/sdd`, not NVMe. Storage layout TBD pending HAT bring-up.
- `pi-03` and `pi-04` (Pi 4) use SD or USB boot TBD — no NVMe HAT available for Pi 4.
- `pi-fw` retains SD (stateless, no NVMe HAT required).

**Switch automation:** MikroTik CRS310 runs full RouterOS. VLAN configuration is automated via the `community.routeros` Ansible collection alongside the Pi playbooks.

---

## Network Architecture

### Design Goals
- Fully self-contained behind any upstream NAT — works at home, works anywhere
- No dependency on a static IP or specific upstream router
- Outbound-only public exposure via Cloudflare Tunnel (no port forwarding required)
- Remote admin and inter-node connectivity via Tailscale mesh
- Portable: unplug, move, replug — everything reconnects automatically

### VLAN Design

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 10 | Internal | `10.42.0.0/24` | All Pi nodes — services, SSH, monitoring |
| 99 | Uplink | DHCP from upstream | `pi-fw` WAN face only |

The managed switch runs:
- **Access ports** on VLAN 10: `pi-01`, `pi-02`, `pi-03`, `pi-04`
- **Trunk port** to `pi-fw`: carries VLAN 10 (tagged) + VLAN 99 (untagged/native)

### IP Addressing (VLAN 10 — Static)

| Node | IP |
|------|----|
| `pi-fw` | `10.42.0.1` (gateway) |
| `pi-01` | `10.42.0.11` |
| `pi-02` | `10.42.0.12` |
| `pi-03` | `10.42.0.13` |
| `pi-04` | `10.42.0.14` |

`pi-fw` runs **dnsmasq** for DHCP + internal DNS (`pi-01.lab` → `10.42.0.11`, etc.).

### Firewall Pi (`pi-fw`) Responsibilities

- **nftables** — stateful firewall, inter-VLAN routing, NAT from VLAN 10 → VLAN 99
- **dnsmasq** — DHCP for VLAN 10, local DNS resolver for `*.lab` short names
- **tailscale** — subnet router advertising `10.42.0.0/24`; all nodes reachable remotely without installing Tailscale per-node
- **cloudflared** — single Cloudflare Tunnel instance, ingress rules proxy public subdomains to internal service ports

All four are lightweight enough for 2GB RAM. No containers on `pi-fw`.

### Tailscale Topology

```
Remote client (phone / laptop)
        │  Tailscale mesh
        ▼
    pi-fw  (subnet router — advertises 10.42.0.0/24)
        │
  ┌─────┴────────────────────────────────┐
  │          VLAN 10  (10.42.0.0/24)     │
  │   pi-01 / pi-02 / pi-03 / pi-04     │
  └──────────────────────────────────────┘
```

Service nodes do not need Tailscale installed. `pi-04` (Ansible control) is the one exception — install Tailscale there directly so Ansible can reach all nodes via Tailscale even when `pi-fw` is being reconfigured.

### Cloudflare Tunnel Topology

```
Internet user → Cloudflare edge (bitrot.me)
                        │  outbound tunnel
                        ▼
                 pi-fw (cloudflared)
                        │  HTTP to internal service IPs
                        ▼
            pi-01 / pi-02 / pi-03 (service ports)
```

`cloudflared` ingress rules map subdomains to internal service URLs. TLS terminated at Cloudflare; internal traffic is plain HTTP on the private VLAN. No reverse proxy (Caddy/Nginx) needed on service nodes.

---

## Service Assignment

### `pi-01` — Comms

| Service | Image | Port | Public |
|---------|-------|------|--------|
| Mattermost | `mattermost/mattermost-team-edition` | 8065 | Yes — `chat.bitrot.me` |
| Stalwart Mail | `stalwartlabs/mail-server` | 8080 (web), 25/465/587/993 | Partial (see note) |
| HomelabBot | custom Python image | internal | No — Tailscale only |
| Postgres (Mattermost) | `postgres:16` | internal | No |

**Mail deliverability note:** Outbound SMTP from a dynamic/residential IP will hit blocklists. Recommended approach:
- **Webmail** (`mail.bitrot.me`) — exposed via Cloudflare Tunnel
- **IMAP/SMTP client access** — Tailscale only, no public exposure
- **Outbound relay** — configure Stalwart to relay through Mailgun free tier (10k/month) or similar

This keeps the full Stalwart stack available while deferring the hard deliverability problem.

### `pi-02` — Data & Sync

| Service | Image | Port | Public |
|---------|-------|------|--------|
| Nextcloud | `nextcloud:apache` | 80 | Yes — `cloud.bitrot.me` |
| Miniflux | `miniflux/miniflux` | 8080 | Yes — `rss.bitrot.me` |
| Syncthing | `syncthing/syncthing` | 8384 (UI), 22000 | No — Tailscale only |
| Postgres (Nextcloud) | `postgres:16` | internal | No |
| Postgres (Miniflux) | `postgres:16` | internal | No |
| Redis (Nextcloud) | `redis:alpine` | internal | No |

### `pi-03` — Utility, Monitoring & Blog

| Service | Image | Port | Public |
|---------|-------|------|--------|
| Kavita | `kizaing/kavita` | 5000 | Yes — `books.bitrot.me` |
| Uptime Kuma | `louislam/uptime-kuma` | 3001 | Yes — `status.bitrot.me` |
| Homepage | `ghcr.io/gethomepage/homepage` | 3000 | No — Tailscale only |
| Blog (nginx) | `nginx:alpine` | 8090 | Yes — `bitrot.me` (apex, no Access gate) |

Hugo runs natively on pi-03 (not in a container). A systemd timer polls the git repo every 5 minutes and rebuilds to `/data/blog/public/` when new commits are detected. nginx serves the static output.

### `pi-04` — Control & Spare

- **Ansible control node** — runs playbooks against the rest of the lab
- **Tailscale** installed directly (so it can reach nodes even during `pi-fw` maintenance)
- No production services initially; available for future expansion or as a hot spare

---

## Storage Layout

Each service node has a local NVMe for all persistent data. No NFS, no shared storage dependencies.

### `pi-01` NVMe layout
```
/data/
  mattermost/     # Mattermost data + attachments
  postgres/       # Mattermost Postgres data dir
  stalwart/       # Stalwart mail store + config
```

### `pi-02` NVMe layout
```
/data/
  nextcloud/      # Nextcloud data directory
  postgres/       # Nextcloud + Miniflux Postgres data dirs
  syncthing/      # Syncthing root
```

### `pi-03` NVMe layout
```
/data/
  kavita/         # Kavita library
  uptime-kuma/    # Uptime Kuma data
```

Docker Compose volumes bind-mount from `/data/` so paths are predictable and easy to back up.

---

## Backup Strategy

Each node backs up its own data independently. No aggregation node required.

| Node | What | How | Destination | Schedule |
|------|------|-----|-------------|----------|
| `pi-01` | `/data/` | Restic snapshot | Backblaze B2 | Daily |
| `pi-02` | `/data/` | Restic snapshot | Backblaze B2 | Daily |
| `pi-03` | `/data/` | Restic snapshot | Backblaze B2 | Daily |
| all nodes | Postgres DBs | `pg_dump` → `/data/backups/` before Restic runs | B2 (via Restic) | Daily |
| all nodes | Compose configs | Git (this repo) | GitHub | On change |

A `restic check` systemd timer runs weekly on each node and pushes a heartbeat to Uptime Kuma's push endpoint. Missing heartbeat = alert.

B2 bucket structure: one bucket per node (`pi-homelab-pi-01`, etc.) for clean isolation and independent restore.

---

## DNS (bitrot.me — Cloudflare)

All public subdomains are Cloudflare-proxied (orange cloud). Cloudflare Tunnel routes traffic — no A record pointing to a home IP.

| Subdomain | Service | Access |
|-----------|---------|--------|
| `cloud.bitrot.me` | Nextcloud | Public |
| `chat.bitrot.me` | Mattermost | Public |
| `mail.bitrot.me` | Stalwart webmail | Public |
| `rss.bitrot.me` | Miniflux | Public |
| `books.bitrot.me` | Kavita | Public |
| `status.bitrot.me` | Uptime Kuma | Public |
| `bitrot.me` | Blog (Hugo/nginx) | Public — no Access gate |

Internal services (Homepage, Syncthing UI, SSH) are reachable via Tailscale using `*.lab` short names resolved by `pi-fw` dnsmasq.

---

## Portability Checklist

When physically moving the lab to a new location:

1. Plug `pi-fw` WAN port into new upstream router — obtains DHCP lease automatically
2. Tailscale re-establishes mesh automatically
3. Cloudflare Tunnel re-establishes automatically (outbound-only)
4. All services come up via Docker Compose `restart: unless-stopped` on boot
5. Update nothing — no IPs, DNS records, or firewall rules depend on the upstream network

The only assumption is that the upstream router provides a DHCP lease and outbound internet access.

---

## Container Runtime

**Podman Quadlets** (no Docker). Each container is a `.container` systemd unit under `/etc/containers/systemd/`. Multi-container services (Mattermost+Postgres, Nextcloud+Postgres+Redis, Miniflux+Postgres) share named Podman networks defined via `.network` unit files. Lifecycle is managed entirely by systemd — no daemon, no socket.

`pi-fw` runs no containers (native systemd units only). Pi 4 nodes (`pi-03`, `pi4-01`, `pi4-02`) mount `/data` from pi-nas over NFS rather than local block storage.

---

## Resolved Decisions

- **pi-04 role:** Ansible control node + hot spare. No production services initially.
- **Stalwart outbound relay:** Mailgun free tier (10k/month, portable, no dependency on x86 lab).
- **Nextcloud deployment:** Standard `nextcloud:apache` + Postgres + Redis in Compose. Not AIO — AIO's bundled Collabora/Talk adds RAM pressure on Pi 5 and has friction with Cloudflare Tunnel's TLS handling. Collabora can be added later as a separate container if needed.
- **Uptime Kuma:** Expose `status.bitrot.me` via Cloudflare Tunnel (same pattern as `status.hrkr.io` on the x86 lab).
- **Cloudflare Access:** Enabled for all public subdomains. Free tier (50 users), zero-trust auth gate at Cloudflare edge before traffic reaches the tunnel.
- **Node storage:** Boot from NVMe on all service nodes — no SD cards on `pi-01` through `pi-04`. `pi-fw` retains SD (stateless, no NVMe hat).

---

## Implementation Phases

### Phase 0 — Hardware & Network
- Flash Pi OS 64-bit to all nodes
- Configure managed switch VLANs (VLAN 10 internal, VLAN 99 uplink trunk on `pi-fw`)
- Set static IPs on all nodes
- Stand up `pi-fw`: enable IP forwarding, nftables rules, dnsmasq

### Phase 1 — Connectivity
- Install Tailscale on `pi-fw` (subnet router, advertise `10.42.0.0/24`) and `pi-04` (direct)
- Create Cloudflare account for bitrot.me, point nameservers to Cloudflare
- Create Cloudflare Tunnel, install and configure `cloudflared` on `pi-fw`
- Verify remote SSH to all nodes via Tailscale from `pi-04`

### Phase 2 — Storage
- Mount NVMe drives on `pi-01`, `pi-02`, `pi-03` (format ext4 or btrfs, mount at `/data`)
- Configure B2 buckets, test Restic init + first snapshot on each node
- Verify restore on one node before proceeding

### Phase 3 — Services: pi-02 first
- Deploy Nextcloud + Miniflux + Syncthing on `pi-02`
- Configure `cloudflared` ingress rules for `cloud.bitrot.me`, `rss.bitrot.me`
- Validate public access and Nextcloud file sync from a client

### Phase 4 — Services: pi-01
- Deploy Stalwart + Mattermost + HomelabBot on `pi-01`
- Configure Stalwart outbound relay
- Add `chat.bitrot.me`, `mail.bitrot.me` to `cloudflared` ingress

### Phase 5 — Services: pi-03
- Deploy Kavita + Uptime Kuma + Homepage on `pi-03`
- Load Kavita library, configure Uptime Kuma monitors + Restic push heartbeats
- Add `books.bitrot.me` to `cloudflared` ingress

### Phase 6 — Ansible
- Write Ansible inventory and playbooks on `pi-04` for all nodes
- Codify `pi-fw` config (nftables, dnsmasq, cloudflared, Tailscale) as Ansible roles
- Codify per-node Docker Compose deployment as Ansible roles

### Phase 7 — Hardening & Validation
- Cloudflare WAF rules for public subdomains
- nftables inter-VLAN lockdown (restrict to only required ports)
- Portability test: unplug from UDM Pro, connect to a different router, confirm full recovery
- Document remaining gaps
