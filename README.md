# pi-homelab

Ansible automation for a portable, self-contained Raspberry Pi homelab hosted at
`bitrot.me`. The lab runs entirely behind any upstream network — no static IP, no
port forwarding — using a Cloudflare Tunnel for public ingress and Tailscale for
remote administration. Unplug it, move it, plug it back in, and everything
reconnects on its own.

## Architecture

```
Internet ──▶ Cloudflare edge (bitrot.me) ──┐
                                           │  outbound tunnel (cloudflared)
Remote client ──▶ Tailscale mesh ──┐       │
                                   ▼       ▼
                                 pi-fw  (router / firewall / DNS / tunnel)
                                   │  VLAN 10 — 10.42.0.0/24
        ┌──────────────────┬───────┴────────┬──────────────────┐
      pi-01              pi-02             pi-03              pi-nas
      comms              data              utility            storage
```

`pi-fw` is the only node with a foot in two networks: a WAN interface that takes a
DHCP lease from whatever upstream router it is plugged into, and a LAN interface
that feeds the managed switch. It runs `nftables` (firewall + NAT), `dnsmasq`
(DHCP + internal `.lab` DNS), `tailscale` (subnet router advertising the lab
network), and `cloudflared` (the public tunnel). No containers run on `pi-fw`.

### Nodes

| Node     | Hardware                        | Storage                  | Role                                            |
|----------|---------------------------------|--------------------------|-------------------------------------------------|
| `pi-fw`  | Pi 5 + [dual 2.5GbE HAT](https://docs.radxa.com/en/accessories/network/dual-2.5-router-hat)          | SD                       | Router, firewall, DNS, Tailscale, Cloudflare    |
| `pi-01`  | Pi 5                            | NVMe                     | Comms: Rocket.Chat, HomelabBot                  |
| `pi-02`  | Pi 5                            | NVMe                     | Data: Nextcloud, Miniflux, Syncthing            |
| `pi-03`  | Pi 4                            | NFS from `pi-nas`        | Utility: Kavita, Uptime Kuma, Homepage, blog    |
| `pi-04`  | Pi 4                            | NFS from `pi-nas`        | Ansible control node, spare                     |
| `pi-nas` | Pi 5 + [Radxa Penta SATA HAT](https://docs.radxa.com/en/accessories/storage/penta-sata-hat)     | 4× SATA SSD, btrfs RAID-10 | NFS server for the Pi 4 nodes                  |

A MikroTik CRS310 managed switch enforces the lab VLAN and is configured via
Ansible (`community.routeros`). Each service node drives a small I2C status
display (SSD1306 OLED on the Pi 5/Pi 4 nodes, a 20×4 character LCD on `pi-nas`).

### Networking

- **VLAN 10** (`10.42.0.0/24`) carries all lab traffic. `pi-fw` is the gateway at
  `.1`; service nodes hold static DHCP reservations.
- **Public services** are exposed only through the Cloudflare Tunnel, gated by
  Cloudflare Access. TLS terminates at the Cloudflare edge; traffic inside the lab
  is plain HTTP on the private VLAN.
- **Admin access** (SSH, Syncthing UI, Homepage) is reachable over Tailscale using
  `*.lab` short names resolved by `pi-fw`. SSH is firewalled to the LAN and the
  Tailscale interface only.

## Containers

Services run as **Podman quadlets** — each container is a systemd unit under
`/etc/containers/systemd/`. There is no Docker daemon. `pi-fw` services are native
systemd units rather than containers.

## Repository layout

```
ansible/
  ansible.cfg
  inventory/
    hosts.yml                     # node groups
    group_vars/all/
      user_inputs.yml             # non-secret config (gitignored)
      user_inputs.example.yml     # template
      vault.yml                   # ansible-vault encrypted secrets (gitignored)
      vault.example.yml           # template
    host_vars/                    # per-node addresses, displays, backup targets
  playbooks/
    site.yml                      # full run: bootstrap → foundation → services
    00-bootstrap.yml              # users, SSH hardening, displays, auto-updates
    10-foundation.yml             # mikrotik switch + pi-fw
    20-services.yml               # all service nodes
    mikrotik.yml, pi-fw.yml
    pi-01-comms.yml, pi-02-data.yml, pi-03-utility.yml, pi-04-control.yml, pi-nas.yml
  templates/                      # quadlets, systemd units, service configs
blog/                             # Hugo source for bitrot.me (PaperMod)
plan.md                           # original architecture plan
```

## Usage

The control node is `pi-fw` itself (`ansible_connection: local`). Ansible, the
required collections (`community.routeros`, `ansible.posix`, `community.general`),
and a vault password file at `~/.local/ansible/pi-homelab-vault-pass` are expected
to be present.

```bash
# First-time secrets setup
cp ansible/inventory/group_vars/all/user_inputs.example.yml \
   ansible/inventory/group_vars/all/user_inputs.yml
cp ansible/inventory/group_vars/all/vault.example.yml \
   ansible/inventory/group_vars/all/vault.yml
ansible-vault encrypt ansible/inventory/group_vars/all/vault.yml
# edit both files to fill in CHANGE_ME values

cd ansible

# Provision everything
ansible-playbook playbooks/site.yml

# Or target a layer / node
ansible-playbook playbooks/00-bootstrap.yml
ansible-playbook playbooks/pi-fw.yml
ansible-playbook playbooks/pi-02-data.yml --limit pi-02
```

`user_inputs.yml` and `vault.yml` are gitignored; the `.example.yml` files are the
committed templates. Secrets live only in the encrypted vault.

## Backups

Each service node snapshots `/data` to a single Backblaze B2 bucket with
`restic`, under a per-node path prefix. Database dumps (`pg_dump`, `mongodump`)
run before each snapshot. A systemd timer drives the schedule.

## Documentation

- [docs/services.md](docs/services.md) — per-node service reference (what runs
  where, ports, data paths, public hostname map).
- [docs/runbook.md](docs/runbook.md) — task-oriented operations runbook (backups
  and restore, node recovery, infrastructure changes, service gotchas).
- [plan.md](plan.md) — original architecture plan.

## License

BSD 3-Clause. See [LICENSE](LICENSE).
