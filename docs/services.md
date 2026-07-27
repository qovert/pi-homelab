# Per-Node Service Reference

Quick reference for every node: what runs on it, where its data lives, how it is
reached, and which playbook manages it. All service containers are Podman
quadlets under `/etc/containers/systemd/`; `pi-fw` runs native systemd units.

Addresses are the static VLAN 10 reservations (`10.42.0.0/24`). Public hostnames
are served through the Cloudflare Tunnel on `pi-fw` and gated by Cloudflare
Access (except the blog apex). Internal-only services are reached over Tailscale.

---

## pi-fw — 10.42.0.1

Router, firewall, DNS, and ingress. No containers; everything is native.
Playbooks: `mikrotik.yml`, `pi-fw.yml`.

| Service       | Type    | Listens                        | Exposure                        |
|---------------|---------|--------------------------------|---------------------------------|
| `nftables`    | native  | —                              | Firewall + NAT (LAN→WAN)        |
| `dnsmasq`     | native  | LAN :53/:67, tailscale0 :53    | DHCP + `*.lab.bitrot.me` DNS for the VLAN and Tailscale clients |
| `tailscaled`  | native  | —                              | Subnet router for `10.42.0.0/24`; exposes lab to tailnet |
| `cloudflared` | native  | outbound only                  | Public tunnel for `bitrot.me`   |

- WAN interface (`eth1`) takes DHCP from the upstream router but ignores DHCP-pushed
  DNS. LAN interface (`eth2`) is the gateway for the lab at `10.42.0.1` and feeds the
  MikroTik switch.
- pi-fw uses its own dnsmasq (`127.0.0.1`) for DNS; Tailscale `accept-dns` is disabled
  so MagicDNS cannot overwrite `/etc/resolv.conf`.
- SSH is firewalled to the LAN and `tailscale0` only.
- Cloudflare ingress rules live in `templates/pi-fw/cloudflared-config.yml.j2`.
- **Tailscale split DNS:** to resolve `*.lab.bitrot.me` on Tailscale clients, add a
  custom nameserver for `lab.bitrot.me` pointing to `100.82.17.40` in the Tailscale
  admin console. The subnet route (`10.42.0.0/24`) must also be approved there.

---

## pi-01 — 10.42.0.11 (comms)

NVMe storage. Playbook: `pi-01-comms.yml`.

| Service            | Container          | Port(s)                              | Public FQDN        | Data                         |
|--------------------|--------------------|--------------------------------------|--------------------|------------------------------|
| Rocket.Chat        | `rocketchat`       | 3000                                 | `chat.bitrot.me`   | (in MongoDB)                 |
| MongoDB            | `rocketchat-mongo` | internal (`rocketchat-net`)          | —                  | `/data/rocketchat/mongodb`   |
| HomelabBot         | `homelabbot`       | internal                             | — (deferred)       | `/opt/homelabbot`            |

- MongoDB runs as a single-node replica set (`rs0`); the replica host must be
  `rocketchat-mongo:27017`. Admin login is `ditto` / `ADMIN_PASS` from
  `/opt/rocketchat/rocketchat.env`.
- HomelabBot is not yet built — the quadlet references a locally built image.
- Mail is not self-hosted on this lab — outbound app notifications relay through
  the self-hosted Mailu on hrkr.io's mail-01 (`services.mail`, auth as
  `noreply@bitrot.me`); inbound mail for bitrot.me also lives on mail-01.
  Rocket.Chat's own SMTP settings are set via `OVERWRITE_SETTING_SMTP_*` env
  vars in `rocketchat.env.j2` rather than the admin UI/API (settings changes
  there require the admin's own TOTP step-up, which a deploy can't provide).
- **SSO via Authelia (OIDC)**: registered as client_id `rocketchat` in
  `authelia/configuration.yml.j2`. Custom OAuth service is created in the
  admin UI named `Authelia` (capital A) — but the callback route is keyed off
  the lowercase internal service name, so `redirect_uris` must point at
  `/_oauth/authelia` (lowercase), not `/_oauth/Authelia`. `key_field` must be
  `sub` (or another claim Authelia's userinfo actually returns) — its default
  of `username` doesn't exist as a claim and silently breaks every login with
  `User not found [401]` even though the identity response itself is fine.
  `merge_users: true` matches by email, so an existing account only merges
  correctly if its stored email matches what Authelia/LLDAP asserts for that
  user — otherwise a new, non-admin account gets created instead, with its
  username defaulted to the raw `sub` UUID. `OVERWRITE_SETTING_*` can seed
  these fields on a fresh deploy, but once saved once via the admin UI they
  become "admin-touched" and no longer accept further env-var overrides.

---

## pi-02 — 10.42.0.13 (data)

NVMe storage. Playbook: `pi-02-data.yml`.

| Service           | Container          | Port(s)                | Public FQDN       | Data                         |
|-------------------|--------------------|------------------------|-------------------|------------------------------|
| Nextcloud         | `nextcloud`        | 80                     | `cloud.bitrot.me` | `/data/nextcloud/html`       |
| Nextcloud DB      | `nextcloud-db`     | internal               | —                 | `/data/nextcloud/postgres`   |
| Nextcloud cache   | `nextcloud-redis`  | internal               | —                 | —                            |
| Miniflux          | `miniflux`         | 8081 → 8080            | `rss.bitrot.me`   | (in Postgres)                |
| Miniflux DB       | `miniflux-db`      | internal               | —                 | `/data/miniflux/postgres`    |
| Syncthing         | `syncthing`        | 8384 (UI), 22000, 21027| — (Tailscale)     | `/data/syncthing`            |
| Vaultwarden       | `vaultwarden`      | 8082 → 80              | `vault.bitrot.me` | `/data/vaultwarden`          |
| MeTube            | `metube`           | 8083 → 8081            | — (Tailscale)     | `/data/metube`               |

- Postgres data dirs must be owned by **UID 70** (`postgres:16-alpine`).
- Nextcloud trusts `pi-fw` (10.42.0.1) as a reverse proxy and overwrites protocol
  to HTTPS for correct URL generation behind the tunnel.
- Syncthing UI is internal-only over Tailscale; `22000` is its sync transport.
- MeTube (yt-dlp web UI) is internal-only over Tailscale — it has no built-in
  auth of its own, so it isn't put behind the Cloudflare tunnel like the other
  services here. Downloaded files land in `/data/metube`, owned UID/GID 1000
  to match the container's `UID`/`GID` env vars.

---

## pi-03 — 10.42.0.14 (utility)

Pi 4. `/data` is NFS-mounted from `pi-nas`. Playbook: `pi-03-utility.yml`.

| Service       | Container      | Port(s)      | Public FQDN         | Data                       |
|---------------|----------------|--------------|---------------------|----------------------------|
| Kavita        | `kavita`       | 5000         | `books.bitrot.me`   | `/data/kavita/{library,config}` |
| Uptime Kuma   | `uptime-kuma`  | 3001         | `status.bitrot.me`  | `/data/uptime-kuma`        |
| Homepage      | `homepage`     | 3000         | — (Tailscale)       | `/data/homepage`           |
| Blog (nginx)  | `blog`         | 8090 → 80    | `bitrot.me` (apex)  | `/data/blog/public`        |

- The blog is a Hugo static site (PaperMod theme). A systemd timer
  (`blog-build`) polls the source repo and rebuilds into `/data/blog/public`;
  nginx serves the output. Source is cloned to `/opt/blog/source`.
- Homepage is the internal dashboard; not exposed publicly. Its container
  image (`ghcr.io/gethomepage/homepage`) validates the Host header and
  rejects anything not in `HOMEPAGE_ALLOWED_HOSTS` — every hostname/IP it's
  reached by (including the `pi-03.lab.bitrot.me` wildcard-DNS name) needs to
  be listed there or requests fail with "Host validation failed" in the logs
  and a blank page in the browser.
- Homepage's config dir is on NFS (root_squash) — the image defaults to
  running as root, which gets squashed to an anonymous UID with no access to
  the admin-owned (750) config dir. Pinned to `User=1000:1000` in the
  quadlet to match, same pattern as Uptime Kuma.
- The blog apex (`bitrot.me`) is the only public hostname not behind Access.
- **This node's NFS mount from pi-nas took kavita/homepage/blog down for over
  a month (2026-06-13 to 2026-07-27)** after a mass reboot where pi-nas
  wasn't up yet — the one-shot mount attempt failed and nothing ever
  retried. Mitigated with `x-systemd.automount` (see pi-nas above for the
  `_netdev` gotcha), confirmed to survive a real reboot, but this fix was
  not exhaustively stress-tested against every possible boot-race timing —
  worth a closer look if it recurs.

---

## pi-05 — 10.42.0.16 (spare)

Pi 4. `/data` is NFS-mounted from `pi-nas`. Bootstrapped only (base
hardening, NFS mount, I2C display) — no service role assigned yet.
Playbook: `00-bootstrap.yml` (`pi4_nodes` group play covers it, despite the
naming mismatch — see note below).

- Named `pi-05` rather than `pi4-01`/`pi4-02` (the other reserved-but-unused
  spare slots) because this one won't stay role-less for long; it's grouped
  under `pi4_nodes` in `hosts.yml` anyway since that play (generic NFS-mount
  bootstrap, no fixed service) is exactly what an unassigned node needs
  regardless of its final name.
- MAC-verified: `dc:a6:32:6b:ac:77`.

---

## pi-nas — 10.42.0.12 (storage)

Pi 5 + Radxa Penta SATA HAT, 4× SATA SSD. Playbook: `pi-nas.yml`.

| Service            | Type   | Detail                                                |
|--------------------|--------|-------------------------------------------------------|
| btrfs RAID-10      | native | 4 SSDs, label `pi-nas-data`, mounted at `/data`       |
| `nfs-kernel-server`| native | Exports `/data/nfs/{pi-03,pi4-01,pi4-02,pi-05}`       |

- The array is created with `--sectorsize 16384` to match the Pi 5 page size.
- NFS exports back the Pi 4 nodes, which have no local bulk storage.
- `_netdev` must **not** be combined with `x-systemd.automount` on the client
  mount options (see pi-03 above) — `_netdev` alone pulls the mount into
  `remote-fs.target`'s eager boot-time chain regardless, defeating the whole
  point of the automount indirection.

---

## pi-04 — 10.42.0.15 (control / identity)

Pi 4. Runs the identity stack on **local** storage (SD), not NFS, so
authentication does not depend on `pi-nas`. Playbooks: `pi-04-identity.yml`
(+ control setup in `00-bootstrap.yml`).

| Service      | Container  | Port(s)      | Public FQDN       | Data              |
|--------------|------------|--------------|-------------------|-------------------|
| Authelia     | `authelia` | 9091         | `auth.bitrot.me`  | `/data/authelia`  |
| LLDAP        | `lldap`    | 17170 (UI), 3890 (LDAP, internal) | — (Tailscale) | `/data/lldap` |
| Ansible      | native     | —            | —                 | —                 |
| `tailscaled` | native     | —            | —                 | —                 |

- **Authelia** is the OIDC provider / authentication portal; **LLDAP** is the
  user directory it authenticates against (`ldap://lldap:3890` on the
  `identity-net` container network). LDAP base DN is `dc=bitrot,dc=me`.
- `auth.bitrot.me` is public via the tunnel but **ungated** — it powers the
  Cloudflare Access gate, so it cannot sit behind it (same exception as the blog).
- LLDAP's admin UI (`:17170`) is internal-only over Tailscale.
- Identity DBs are SQLite; restic backs them up via consistent `.backup` dumps
  before each snapshot.
- `pi-04` is also the fallback Ansible control node and a direct tailnet member,
  so the lab stays reachable during `pi-fw` maintenance.

---

## Displays

Each service node drives a small I2C status display showing hostname, IP, and
load. Configured per-host in `host_vars` (`display:` block) and deployed by
`00-bootstrap.yml`.

| Node              | Display              | I2C address |
|-------------------|----------------------|-------------|
| pi-01 … pi-05     | SSD1306 OLED 128×64  | `0x3C`      |
| pi-nas            | Character LCD 20×4   | `0x27`      |

---

## Public hostname → backend map

Configured in `templates/pi-fw/cloudflared-config.yml.j2`.

| Hostname            | Backend                | Access gate |
|---------------------|------------------------|-------------|
| `bitrot.me`         | `pi-03:8090` (blog)    | none        |
| `auth.bitrot.me`    | `pi-04:9091` (Authelia)| none (is the IdP) |
| `cloud.bitrot.me`   | `pi-02:80`             | Access      |
| `chat.bitrot.me`    | `pi-01:3000`           | Access      |
| `rss.bitrot.me`     | `pi-02:8081`           | Access      |
| `books.bitrot.me`   | `pi-03:5000`           | Access      |
| `status.bitrot.me`  | `pi-03:3001`           | Access      |
| `vault.bitrot.me`   | `pi-02:8082`           | Access      |
