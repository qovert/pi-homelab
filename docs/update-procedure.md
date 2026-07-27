# Fleet Update Procedure

Task-oriented plan for running OS package updates and container image updates
across the lab. Written up after a full pass on 2026-07-27 that took much
longer than expected because several reboot-triggered regressions only
surfaced by actually rebooting each node — see **Caveats** at the end before
running this.

This lab is a portable demo/testing rig, not mission-critical — the bar is
"comes back up cleanly after a reboot," not zero-downtime. Reboots are fine;
budget time for the verification steps below, since skipping them is exactly
how the regressions in the caveats section went unnoticed the first time.

## Scope

Covers the six Pi nodes (`pi-fw`, `pi-01`, `pi-02`, `pi-03`, `pi-04`, `pi-05`,
`pi-nas`). **Does not cover the MikroTik switch** — RouterOS upgrades are a
different mechanism (firmware upload + reboot) with a much larger blast
radius (a bad switch reboot cuts LAN connectivity to everything, not just one
node) and deserve their own, separate, more careful pass.

## Order

```
pi-nas → pi-02 → pi-03 → pi-01 → pi-04 → pi-fw
```

Rationale:
- **pi-nas first** — pi-03 and pi-05 mount `/data` from it over NFS; get it
  current and confirmed healthy before anything depends on it.
- **pi-02, pi-03, pi-01, pi-04** — order among these four doesn't matter
  much; grouped apps-before-identity here mostly by convention. `pi-03`
  needs `pi-nas` already up (NFS mount); the others are independent.
- **pi-fw last, always.** Rebooting it temporarily breaks the Tailscale
  subnet route to every other `10.42.0.0/24` node — reachable again once it
  finishes rebooting and `tailscaled`/`cloudflared` re-establish, but doing
  it last means every other node's update is already confirmed done before
  you lose the ability to reach them easily.

## Per-node procedure

Repeat for each node in order:

```bash
ssh admin@<node-ip>

# 1. OS packages
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y

# 2. Check whether a new kernel needs a reboot
uname -r   # compare against the newest linux-image-* package just installed

# 3. Pull fresh images for every container on this node
sudo podman ps --format '{{.Names}}: {{.Image}}'
sudo podman pull <image>   # for each one

# 4. Baseline health check BEFORE rebooting
curl -sf -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:<port>   # for each service

# 5. Reboot (covers new kernel + restarts every quadlet onto the freshly pulled images)
sudo reboot
```

Then, once it's back:

```bash
# 6. Confirm the new kernel is actually running
uname -r

# 7. Confirm every container came back and is healthy
sudo podman ps --format '{{.Names}}: {{.Status}}'
curl -sf -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:<port>   # for each service

# 8. Confirm no packages were left pending
sudo apt list --upgradable 2>/dev/null | grep -v '^Listing'   # should be blank
```

### Node-specific extras

- **pi-nas**: also check the btrfs array before and after —
  `sudo btrfs filesystem show /data` and `sudo btrfs device stats /data`
  (all-zero error counters) — and confirm exports survived:
  `sudo /usr/sbin/exportfs -v`.
- **pi-03** (and any NFS-mounted spare, e.g. `pi-05`): explicitly confirm
  `/data` is mounted after the reboot, not just that containers are up —
  `mount | grep /data`. This is the node most likely to be affected by the
  NFS boot-race caveat below.
- **pi-fw**: after it's back, confirm `cat /proc/sys/net/ipv4/ip_forward`
  is `1` (not just that the service is "active" — see caveat below), then
  confirm the LAN is actually reachable again from wherever you're running
  this from: `ssh admin@10.42.0.11 hostname` (or any other node) should
  succeed. Also check `sudo tailscale status` for the IP-forwarding
  health-check warning line — it disappears once forwarding is genuinely on.

## The one rule that matters most

**Never report a reboot-persistence fix as done without an actual reboot
test.** Config that reads correctly on live inspection can still silently
fail to survive a real reboot — this happened three separate times in one
update pass (NFS automount, IP-forwarding sysctl, a cloud-init hostname
fix), and each time the only way to actually know was to reboot again and
check. If a fix can't be reboot-tested in the moment, say so explicitly
rather than implying it's confirmed.

## Caveats / known gotchas going into the next update pass

- **NFS mounts**: `x-systemd.automount` combined with `_netdev` does **not**
  give you a mount that safely waits for the network — `_netdev` alone still
  pulls the mount into `remote-fs.target`'s eager boot chain regardless,
  reproducing the exact boot race it's meant to avoid. Use
  `x-systemd.automount` **without** `_netdev`. This was fixed and verified
  with one real reboot on `pi-03`, but has not been exhaustively stress-
  tested against every possible timing — worth a closer look if
  kavita/homepage/blog go down again after a reboot.
- **IP forwarding on pi-fw**: must be set via `sysctl_file:
  /etc/sysctl.d/99-ip-forward.conf` in the `ansible.posix.sysctl` task, not
  the module's default `/etc/sysctl.conf` — current systemd-sysctl silently
  stops applying that legacy path at boot. If this task ever gets "cleaned
  up" back to the default, the whole LAN loses tailnet routing on the next
  `pi-fw` reboot with no obvious cause.
- **Raspberry Pi Imager / cloud-init hostname reversion**: if a card was
  imaged by reusing a saved Imager profile from a different device, its
  `/boot/firmware/user-data` seed can silently re-apply the wrong hostname
  on **every** boot via `cc_update_hostname` (frequency "always"). Confirmed
  to actually happen to `pi-04`. Always set the hostname field explicitly
  when flashing, and if a node's hostname mysteriously keeps reverting,
  check the seed file, not just `/etc/hostname`.
- **Workstation config drift**: `user_inputs.yml` and `vault.yml` are
  gitignored and edited directly on `pi-fw`. Any other checkout's copies can
  silently go stale or (until 2026-07-27) even fail to exist, and until that
  date a missing `user_inputs.yml` would silently fall back to
  `inventory/examples/user_inputs.example.yml`'s placeholder values with no
  warning at all (fixed by moving the `.example.yml` files out of
  `group_vars/all/`, where Ansible's group_vars loading no longer sees
  them). Diff against pi-fw's real copies before trusting a workstation
  checkout, especially before running any `--limit`-targeted playbook.
- **`playbooks/pi-fw.yml` only works run locally on pi-fw** — it targets a
  host with `ansible_connection: local`, so running it from any other
  machine tries to `sudo` on *that* machine instead and fails outright.
- **Rocket.Chat's `OVERWRITE_SETTING_*` env vars only seed untouched
  defaults.** Once a setting has been saved via the admin UI (or API) even
  once, it's considered admin-touched and further env-var overrides are
  silently ignored on restart — relevant if a future update pass needs to
  change Rocket.Chat's SMTP or OIDC config again; those specific settings
  may need a manual UI correction instead of just redeploying.
- **Vaultwarden's env-file deploy task depends on `vault.vaultwarden`
  existing** in `vault.yml`. If it's missing (stale workstation checkout —
  see above), the whole `pi-02-data.yml` playbook run fails partway through
  on `Deploy Vaultwarden env file`, before ever reaching later tasks in that
  play (this blocked a MeTube deploy once already). `--start-at-task` past
  it is a reasonable workaround if you don't want to fix the vault gap
  right then, but the real fix is syncing the vault.
- **`/data/kavita`'s parent directory is root-owned** (pre-existing, likely
  from Podman auto-creating the bind-mount source before Ansible ever ran
  there) while `/data/kavita/library` underneath it is `admin`-owned. This
  has intermittently caused the `Create Kavita library directory on NFS`
  task to fail with a permission error that looks NFS-attribute-cache-
  related — it hasn't reproduced consistently enough to fully root-cause.
  If it fails, retrying the same task usually succeeds.
- **HomelabBot is expected to crash-loop** — its quadlet references a
  locally-built image that doesn't exist yet (`localhost/homelabbot:latest`).
  Not a regression from an update; leave it alone until the image is
  actually built.
- **The MikroTik switch is out of scope for this procedure** (see Scope,
  above) — don't fold it into a routine Pi update pass.
