# Operations Runbook

Task-oriented procedures for running the lab. For the lookup table of what runs
where, see [services.md](services.md).

**Conventions**
- The Ansible control node is `pi-fw` itself (`ansible_connection: local`); run
  `ansible-playbook` from `~/pi-homelab/ansible` on `pi-fw`.
- Vault password file: `~/.local/ansible/pi-homelab-vault-pass`.
- Every service container is a Podman quadlet — manage it as a systemd unit
  (`systemctl`), never with `podman run`/`podman rm` directly, or the next
  `daemon-reload` will fight you.
- Node addresses: `pi-fw` .1, `pi-01` .11, `pi-nas` .12, `pi-02` .13, `pi-03` .14,
  `pi-04` .15 on `10.42.0.0/24`.

---

## Routine operations

### Restart a service

```bash
sudo systemctl restart <service>          # e.g. nextcloud, rocketchat, miniflux
systemctl status <service> --no-pager
sudo podman ps --format '{{.Names}}: {{.Status}}'
```

Quadlet units are generated from the `.container` files in
`/etc/containers/systemd/`. After editing one of those by hand (don't — edit the
template and re-run the playbook), reload with `sudo systemctl daemon-reload`.

### Deploy a configuration change

```bash
# on your workstation: edit templates / vars, commit, push
cd ~/pi-homelab && git pull          # on pi-fw
cd ansible
ansible-playbook playbooks/pi-02-data.yml --limit pi-02
```

Playbooks are idempotent. A template change notifies the matching restart
handler automatically.

### Check service health

```bash
for n in 11 13 14; do echo "=== 10.42.0.$n ==="; \
  ssh admin@10.42.0.$n "sudo podman ps --format '{{.Names}}: {{.Status}}'"; done
```

---

## Backup & restore

Each service node snapshots `/data` to Backblaze B2 (`b2:pi-homelab:<node>`)
via `restic`, driven by the `restic-backup.timer` systemd timer. Database dumps
run first: `pg_dump` for Postgres services, `mongodump` for Rocket.Chat.

### Run a backup now / check status

```bash
sudo /usr/local/bin/restic-backup            # manual run (also inits the repo)
systemctl status restic-backup.timer --no-pager
sudo journalctl -u restic-backup -n 30
```

### List snapshots

Credentials live in `/etc/restic/env` (root-only). Load them into the current shell:

```bash
# on the node, as root
sudo bash -c 'set -a; source /etc/restic/env; restic snapshots'
```

### Restore files

```bash
sudo bash -c 'set -a; source /etc/restic/env; \
  restic restore latest --target /tmp/restore --include /data/nextcloud'
```

Restore into a scratch path first, inspect, then move into place. Stop the
affected service before overwriting live data.

### Restore a Postgres database

Dumps live at `/data/<service>/backups/<db>.sql` inside each snapshot. To load
one into a running DB container (UID 70, `postgres:16-alpine`):

```bash
sudo systemctl stop nextcloud                       # stop the app, keep the DB
sudo podman exec -i nextcloud-db \
  psql -U nextcloud -d nextcloud < /data/nextcloud/backups/nextcloud.sql
sudo systemctl start nextcloud
```

### Restore the Rocket.Chat database

```bash
# mongodump output is under /data/rocketchat/mongodb/backups inside the snapshot
sudo podman exec -i rocketchat-mongo \
  mongorestore --drop --dir /data/db/backups
sudo systemctl restart rocketchat
```

---

## Node recovery

### Reflash and re-provision a service node

1. Flash Raspberry Pi OS 64-bit with rpi-imager, preloading the `admin` user and
   the lab SSH public key (so first contact is key-based, no `pi` user).
2. Confirm it picks up its DHCP reservation (MAC → IP is pinned in
   `templates/pi-fw/dnsmasq.conf.j2`; new hardware needs its MAC updated there).
3. From `pi-fw`:
   ```bash
   ansible-playbook playbooks/00-bootstrap.yml --limit <node>
   ansible-playbook playbooks/pi-02-data.yml  --limit <node>   # its service play
   ```
4. Restore `/data` from restic if the storage was lost (see above).

**Storage split:** `pi-01`/`pi-02` boot from and store on local NVMe. `pi-03`/`pi-04`
are Pi 4s with no bulk storage — they mount `/data` over NFS from `pi-nas`, so a
reflash there only needs the OS; `/data` is intact on the NAS.

### Recover the pi-nas btrfs array

The array is `raid10` labeled `pi-nas-data`, mounted at `/data`.

```bash
sudo btrfs filesystem show /data        # confirm all 4 devices present
sudo btrfs device stats /data           # check for errors
```

Replace a failed disk:

```bash
sudo btrfs replace start <devid|olddev> /dev/sdX /data
sudo btrfs replace status /data
```

If recreating from scratch, the array **must** be made with the Pi 5 page size or
it will not mount:

```bash
mkfs.btrfs -f --sectorsize 16384 -d raid10 -m raid10 -L pi-nas-data \
  /dev/sda /dev/sdb /dev/sdc /dev/sdd
```

After the array is back, re-run `ansible-playbook playbooks/pi-nas.yml` to restore
the mount and NFS exports, then remount `/data` on the Pi 4 nodes.

### Recover pi-fw

`pi-fw` is stateless (SD card, no `/data`). Reflash, restore
`user_inputs.yml`/`vault.yml` and the vault password file, then:

```bash
ansible-playbook playbooks/10-foundation.yml   # mikrotik + pi-fw
```

The Cloudflare tunnel and Tailscale re-establish outbound automatically once
`cloudflared` and `tailscaled` are back. Re-run `tailscale up
--advertise-routes=10.42.0.0/24` and re-approve the route if the node is new to
the tailnet.

---

## Infrastructure changes

### Add a new public service

1. **Quadlet + playbook:** add `templates/<svc>/<svc>.container.j2` (and a
   `.network` / DB container if needed) and tasks in the node's service playbook.
   Create `/data/<svc>` with the container's expected UID.
2. **Ingress:** add a hostname block to
   `templates/pi-fw/cloudflared-config.yml.j2` pointing at the node IP and port.
3. **Vault/vars:** put the FQDN under `services.<svc>` in `user_inputs.yml` and any
   secrets in `vault.yml`.
4. **DNS:** in Cloudflare, add the subdomain as a tunnel (CNAME →
   `<tunnel-id>.cfargotunnel.com`, proxied).
5. **Access:** create a self-hosted Access application for the hostname with the
   shared policy (skip this for anything intended to be public).
6. Deploy:
   ```bash
   ansible-playbook playbooks/<node>.yml --limit <node>
   ansible-playbook playbooks/pi-fw.yml          # push new ingress
   ```

### Change the MikroTik switch / VLAN

Edit `playbooks/mikrotik.yml` (uses `community.routeros`) and re-run
`ansible-playbook playbooks/mikrotik.yml`. The switch management IP is
`10.42.0.253`.

### Rotate a secret

```bash
ansible-vault edit ansible/inventory/group_vars/all/vault.yml
# then re-run the playbook(s) that consume it
```

Secrets only ever live in the encrypted vault; `user_inputs.yml` and `vault.yml`
are gitignored, with `.example.yml` templates committed.

---

## Service-specific gotchas

### Postgres (Nextcloud, Miniflux)

`postgres:16-alpine` runs as **UID 70** — the `/data/<svc>/postgres` directory and
its contents must be owned `70:70`, or the DB fails with
`could not open file ... Permission denied` after a redeploy. The playbooks set
this correctly; don't reset it to www-data (33) or 999.

### Rocket.Chat / MongoDB

- MongoDB is a single-node replica set `rs0`. The member host must be
  `rocketchat-mongo:27017` (a container-network name), not `localhost`, or
  Rocket.Chat loops on `MongoTopologyClosedError: Topology is closed`.
- If the replica set is uninitialised after a fresh Mongo volume:
  ```bash
  sudo podman exec rocketchat-mongo mongosh --eval \
    "rs.initiate({_id:'rs0',members:[{_id:0,host:'rocketchat-mongo:27017'}]})"
  ```
- Admin login: `ditto` / `ADMIN_PASS` from `/opt/rocketchat/rocketchat.env`.

### Mail

This lab does **not** run a mail server. A dynamic residential IP behind a
Cloudflare Tunnel can't host one (no reachable port 25, no clean rDNS). Instead,
both outbound app notifications **and** inbound mail for `bitrot.me` are handled
by the self-hosted **Mailu on hrkr.io's mail-01** (static Hetzner IP, real
deliverability). Apps relay via authenticated SMTP submission
(`services.mail` → `mail.hrkr.io:465` implicit TLS, auth as `noreply@bitrot.me`
in `vault.smtp`), sending as `noreply@bitrot.me`. `bitrot.me` is added as a Mailu
domain with its own MX/SPF/DKIM/DMARC so the mail is properly aligned.

To debug a relay problem, test submission directly from a node:
```bash
python3 - <<'EOF'
import smtplib, ssl
s = smtplib.SMTP_SSL("mail.hrkr.io", 465, context=ssl.create_default_context())
s.login("noreply@bitrot.me", "<password>")
s.sendmail("noreply@bitrot.me", ["you@example.com"], "Subject: test\r\n\r\nbody")
EOF
```

### Identity (LLDAP + Authelia, pi-04)

- **LLDAP applies `LLDAP_LDAP_USER_PASS` only on first DB init.** Changing
  `vault.lldap.admin_password` later does **not** update the stored password, so
  Authelia's LDAP bind then fails with `Invalid Credentials (49)` and Authelia
  refuses to boot. To rotate it: change it in the LLDAP UI **and** the vault to
  match, or wipe and re-init (`systemctl stop authelia lldap; rm
  /data/lldap/users.db*; systemctl start lldap`) — the latter loses LLDAP users.
- Verify the admin bind directly:
  ```bash
  ldapwhoami -x -H ldap://lldap:3890 \
    -D uid=admin,ou=people,{{ base_dn }} -w '<password>'   # Result: Success (0)
  ```
- **Authelia's mail notifier startup check is disabled** (`disable_startup_check:
  true`) on purpose — the IdP must boot even if SMTP is down, or a mail outage
  locks you out of every service. Mail (2FA enrollment, password resets) needs
  valid `vault.smtp` submission credentials; if mail is unavailable, set
  `services.mail.notifier: filesystem` to write notifications to
  `/data/authelia/notification.txt` instead of emailing.
- `auth.bitrot.me` is ungated at the tunnel — it is the IdP and cannot sit
  behind the Access gate it powers.

### Blog (Hugo)

- Source repo is cloned to `/opt/blog/source`; the `blog-build` systemd timer
  pulls and rebuilds into `/data/blog/public`, which nginx serves on `:8090`.
- Force a rebuild:
  ```bash
  sudo /usr/local/bin/blog-build
  ```
- The PaperMod theme is a git submodule — clones use `recursive: true` and the
  build script runs `git submodule update --init --recursive`.

### arm64 image availability

Several upstreams don't publish arm64 images or pin oddly. Known cases:
Mattermost has **no** arm64 build (replaced with Rocket.Chat); Kavita moved to
`jvmilazz0/kavita`; Mattermost ships **no** arm64 image at all. Always `podman
pull` a new image on the target before wiring it into a quadlet.
