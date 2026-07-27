# Operations Runbook

Task-oriented procedures for running the lab. For the lookup table of what runs
where, see [services.md](services.md). For OS/container updates across the
fleet, see [update-procedure.md](update-procedure.md).

**Conventions**
- The Ansible control node is `pi-fw` itself (`ansible_connection: local`); run
  `ansible-playbook` from `~/pi-homelab/ansible` on `pi-fw`.
- Vault password file: `~/.local/ansible/pi-homelab-vault-pass`.
- Every service container is a Podman quadlet — manage it as a systemd unit
  (`systemctl`), never with `podman run`/`podman rm` directly, or the next
  `daemon-reload` will fight you.
- Node addresses: `pi-fw` .1, `pi-01` .11, `pi-nas` .12, `pi-02` .13, `pi-03` .14,
  `pi-04` .15, `pi-05` .16 on `10.42.0.0/24`.
- `playbooks/pi-fw.yml` only works when run **locally on pi-fw itself** —
  `host_vars/pi-fw.yml` sets `ansible_connection: local`, so invoking it from
  any other workstation tries to `sudo` on *that* machine instead and fails
  with `sudo: a password is required`. Every other playbook works fine
  remotely.
- `user_inputs.yml` and `vault.yml` are gitignored and edited directly on
  whichever host has them — normally pi-fw. A workstation checkout's copies
  are a point-in-time snapshot with **no sync mechanism**; they silently go
  stale the moment someone edits pi-fw's copy directly (confirmed to
  actually happen: an entire `inventory_hosts` block and a `vaultwarden:`
  vault section were missing from a workstation checkout that had simply
  never been refreshed). Before concluding a variable or vault key doesn't
  exist, diff against pi-fw's real copies first.

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

**Storage split:** `pi-01`/`pi-02` boot from and store on local NVMe. `pi-03`/`pi-04`/
`pi-05` are Pi 4s with no bulk storage — they mount `/data` over NFS from `pi-nas`, so a
reflash there only needs the OS; `/data` is intact on the NAS.

### Add a brand-new node (not just reflashing an existing one)

More steps than reflashing, since nothing is reserved yet:

1. **Reserve an IP**: add `pi_XX: 10.42.0.YY` to `inventory_hosts` in
   `user_inputs.yml` (edit on pi-fw, then copy down to any other checkout —
   see the gitignored-config note above).
2. **Inventory stub**: create `inventory/host_vars/pi-XX.yml`
   (`ansible_host: "{{ inventory_hosts.pi_XX }}"`, `restic_node_name`,
   `restic_postgres_dumps: []`, `compose_projects: []`), and add the host
   under the right group in `inventory/hosts.yml`. A role-less spare node can
   reuse the `pi4_nodes` group's bootstrap play regardless of its name — that
   group's purpose (generic NFS-mount bootstrap, no fixed service) matters
   more than matching its literal name.
3. **NFS export** (if it'll mount `/data` from pi-nas): `pi-nas.yml`'s export-
   directory task and `templates/pi-nas/exports.j2` are **hardcoded** per-host
   (`pi-03`, `pi4-01`, `pi4-02`, ...) — they do not derive from group
   membership. A new node outside that hardcoded list needs an explicit
   addition to both, or it gets no export at all.
4. **Flash**: Raspberry Pi Imager, Edit Settings — set hostname **explicitly**
   for this device, don't reuse a saved profile from another card (see the
   cloud-init hostname gotcha below). Preload the `admin` user and lab SSH key.
5. **Identify on the network**: check `/var/lib/misc/dnsmasq.leases` on pi-fw
   for the new device's DHCP-broadcast hostname/MAC on the dynamic pool
   (`10.42.0.100`–`.200`) before it has a static reservation.
6. **Fill in the stub**: add `dhcp_mac` (and `display:` if it has one) to the
   host_vars file from step 2, using the MAC found in step 5.
7. **Deploy in order**: `pi-fw.yml` (DHCP reservation) → reboot the new node
   to pick up its static IP → `pi-nas.yml` (NFS export, if applicable) →
   `00-bootstrap.yml --limit pi-XX` (hardening, NFS mount, display).

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
`cloudflared` and `tailscaled` are back. Tailscale authentication is manual on a
new node (`tailscale up`); once authenticated the playbook handles
`--advertise-routes` and `--accept-dns=false` automatically. Re-approve the subnet
route (`10.42.0.0/24`) in the Tailscale admin console if the node is new to the
tailnet.

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

### pi-fw (router/firewall)

- **IP forwarding must be set via `/etc/sysctl.d/`, not the default
  `/etc/sysctl.conf`.** `ansible.posix.sysctl`'s default `sysctl_file` is the
  legacy `/etc/sysctl.conf` path, which current systemd-sysctl no longer
  applies at boot — confirmed live: the value was correctly present in that
  file but read back as `0` after a real reboot, silently breaking all LAN
  subnet routing until a manual `sysctl --system`. The `Enable IP forwarding`
  task in `pi-fw.yml` sets `sysctl_file: /etc/sysctl.d/99-ip-forward.conf`
  explicitly for exactly this reason — don't revert it to the default.
- `sudo tailscale status` prints a health-check line about IP forwarding
  being disabled when this regresses — that's the fastest way to notice it,
  since the symptom otherwise is just "the whole LAN is unreachable from the
  tailnet" with no obvious cause.

### Raspberry Pi Imager / cloud-init (any node)

- Raspberry Pi OS ships `cloud-init` + `rpi-cloud-init-mods`. Imager's "Edit
  Settings" writes a NoCloud seed to `/boot/firmware/user-data`, and
  `cc_update_hostname` runs with frequency **"always"** — meaning it silently
  re-applies whatever hostname is in that seed file **on every single boot**,
  overwriting any later manual/Ansible hostname fix. If a card gets imaged by
  reusing a saved Imager profile from a different device, the wrong hostname
  (and only the hostname most obviously, but potentially other saved fields
  too) keeps coming back after every reboot. Fix at the source
  (`/boot/firmware/user-data`'s `hostname:` line), not just the live
  `/etc/hostname` — confirmed this actually happened to `pi-04`, which kept
  reporting itself as `pi03` for weeks.
- Always set the hostname field explicitly when flashing a new card, even if
  it feels redundant — don't trust a saved profile from another device.

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

#### SMTP and admin settings in general

- Rocket.Chat's admin settings (SMTP, OAuth, etc.) live in
  `rocketchat_settings` in Mongo, not the container env directly. Check
  current values with:
  ```bash
  sudo podman exec rocketchat-mongo mongosh --quiet rocketchat --eval \
    "db.rocketchat_settings.find({_id: /SMTP/}).forEach(d => print(d._id + ' = ' + JSON.stringify(d.value)))"
  ```
- `OVERWRITE_SETTING_<SettingId>=<value>` env vars (see `Show_Setup_Wizard`
  in `rocketchat.env.j2` for the original example) can seed these — but
  **only while the setting is still at its untouched factory default.** Once
  an admin saves a value via the UI (or API) for that setting, it's
  considered admin-touched and further env-var overrides are silently
  ignored on restart, even though the env var is genuinely present in the
  container (verify with `podman exec rocketchat env | grep ...`). This bit
  us twice: SMTP_* worked cleanly via env var (never touched before), but
  the OAuth Custom fields required manual UI correction after the one-time
  UI creation step, since saving the form once locks them.
- The admin API (`/api/v1/settings/:id`) requires the admin's own **TOTP
  step-up** for privileged settings changes — a deploy has no way to provide
  this, so don't try to script settings changes through the API. Env-var
  overrides or a one-time manual UI action are the only two paths.

#### SSO via Authelia (OIDC)

Registered as `client_id: rocketchat` in `authelia/configuration.yml.j2`.
Custom OAuth service must be created once via **Administration → Settings →
OAuth → Custom OAuth → Add custom OAuth** (there is no way to create a
brand-new Custom OAuth service purely via env var — the setting IDs for it
don't exist until the UI creates them; only *updates* to an already-existing
one can go through `OVERWRITE_SETTING_*`). See `rocketchat.env.j2` for the
exact field values this lab uses. Gotchas found getting this working,
worst first:

- **`key_field` default (`username`) doesn't exist as a claim.** Authelia's
  userinfo response has `sub`, `email`, `preferred_username`, `name` — no
  `username`. Leaving this at its default breaks every login with
  `Exception while invoking method login errorClass [Error]: User not found
  [401]`, even though the identity response Rocket.Chat received was
  perfectly fine (confirm with debug logging, see below). Set it to `sub`.
- **`merge_users: true` matches by email.** If the Rocket.Chat account you
  want SSO to log into has a different stored email than what
  Authelia/LLDAP actually asserts for that identity, merging silently fails
  and a **new, non-admin** account gets created instead — with its username
  defaulted to the raw `sub` UUID (this only happens on fresh account
  creation; an update to an already-matched account applies `username_field`
  normally). Fix the existing account's email to match, then delete the
  spurious duplicate:
  ```bash
  sudo podman exec rocketchat-mongo mongosh --quiet rocketchat --eval "
  db.users.deleteOne({_id: '<duplicate-id>'});
  db.users.updateOne({_id: '<real-account-id>'}, {\$set: {emails: [{address: '<real-email>', verified: true}]}});
  "
  ```
- **Callback URL casing is not what you'd expect.** The custom OAuth service
  as displayed in the admin UI can be named with any casing (e.g.
  `Authelia`), but the actual `/_oauth/<name>` callback route is keyed off
  the **lowercase** internal service identifier, confirmed by testing both
  casings directly and checking which one reached `oauth_server.js` in the
  logs. `redirect_uris` on the Authelia side must use the lowercase form
  regardless of how the service is displayed/named in Rocket.Chat's UI.
- **To actually see what's failing**, bump both sides' logging temporarily
  (revert after): Authelia's `log.level: debug` in `configuration.yml`
  (live-edit `/opt/authelia/configuration.yml` for a quick one-off, or change
  the template for a lasting change) and Rocket.Chat's `Log_Level` setting to
  `"2"` via mongosh, then restart both and retry the login. Authelia's debug
  log shows the full flow (authorization → 1FA → 2FA → consent → token
  exchange → userinfo) and will tell you definitively whether *its* side
  completed successfully; Rocket.Chat's debug log shows the `CustomOAuth`
  "Identity response" it received and the exact exception if the login
  method call then fails.

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

### NFS mounts (pi-03, spare Pi 4 nodes)

`x-systemd.automount` + `_netdev` together **don't** give you a mount that
safely waits for the network — `_netdev` alone still pulls the mount into
`remote-fs.target`'s eager boot-time chain, so a mass reboot where pi-nas
isn't up yet still races and leaves `data.mount` failed (confirmed live,
took kavita/homepage/blog down for over a month on pi-03). Use
`x-systemd.automount` **without** `_netdev`. See services.md's pi-03/pi-nas
sections for the full incident writeup.

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
