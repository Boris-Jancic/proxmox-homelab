# Proxmox Homelab — Ansible

Ansible-managed Proxmox single-node homelab. Replaces an earlier `init-all.sh`
bash script with idempotent playbooks for LXC provisioning and per-service
configuration.

## What this manages

- **LXC containers** — 7 CTs (nginx-proxy, pi-hole, vaultwarden, uptime-kuma,
  homepage, stirling-pdf, plus a `pi-hole-from-scratch` test CT). Creation,
  networking, root SSH key.
- **Pi-hole (v6+)** — installs the package when missing, then enforces the
  web admin password.

Everything else — the Nextcloud-AIO Ubuntu VM, the internals of the other
service CTs — is still manual.

## Layout

```
ansible.cfg                          inventory + roles_path config
inventory.yml                        pve host + lxc_containers + pihole groups
group_vars/all/
  main.yml                           shared vars (template, gateway, ssh pubkey lookup)
  secrets.yml.example                template -> copy to secrets.yml (gitignored)
playbooks/
  provision-lxc.yml                  idempotent CT creation + start
  bootstrap-existing-lxc-keys.yml    one-time SSH key retrofit for pre-Ansible CTs
  configure-pihole.yml               install (if missing) + web admin password
roles/pihole/
  defaults/main.yml                  DNS upstreams, behavior toggles
  tasks/main.yml                     detect -> install -> password orchestrator
  tasks/install.yml                  apt deps, setupVars.conf preseed, unattended installer
  tasks/password.yml                 idempotent `pihole setpassword` via marker file
  templates/setupVars.conf.j2        install-time preseed (v6 migrates to pihole.toml)
requirements.yml                     ansible collection deps
```

## Prerequisites

- Workstation with ansible + `proxmoxer` + `requests`:
  ```
  pip install --user ansible proxmoxer requests
  ansible-galaxy collection install -r requirements.yml
  ```
- An ed25519 SSH key at `~/.ssh/id_ed25519.pub` on whichever machine you run
  Ansible from. Override `controller_ssh_pubkey_path` in
  `group_vars/all/main.yml` if you use rsa or a different file.
- Root SSH access to the PVE node from that same machine.

## Initial setup (one-time)

1. Edit `inventory.yml` — set `pve.ansible_host` to your Proxmox node IP,
   adjust LXC CTIDs / IPs if yours differ.
2. Fill in secrets:
   ```
   cp group_vars/all/secrets.yml.example group_vars/all/secrets.yml
   $EDITOR group_vars/all/secrets.yml
   ```
3. Give the controller passwordless SSH to PVE:
   ```
   ssh-copy-id root@<pve-ip>
   ```

## Workflows

### Provision LXCs

```
ansible-playbook playbooks/provision-lxc.yml
```

Existing CTs report `ok=changed=0` because the module is keyed on `vmid`. New
CTs get created, started, and have the controller pubkey baked into
`/root/.ssh/authorized_keys` at create time.

`--check` is **not** useful here — `community.general.proxmox` skips itself in
check mode. Just run it for real; the module won't mutate existing CTs.

### Retrofit SSH keys into pre-Ansible CTs

For CTs that were created before `pubkey:` was added to the provisioning task.
Idempotent — safe to re-run.

```
ansible-playbook playbooks/bootstrap-existing-lxc-keys.yml
```

Runs against PVE and uses `pct exec` to append the controller pubkey into each
CT's `authorized_keys`.

### Install + configure pi-hole

```
ansible-playbook playbooks/configure-pihole.yml
```

Runs against every host in the `pihole` inventory group. For each host:

- If `/usr/local/bin/pihole` is missing: stages `setupVars.conf`, runs the
  unattended installer, then sets the web password. First run is slow
  (~5–10 min for downloads + install).
- If pi-hole is already installed: jumps straight to the password step.
  Idempotent via the marker file at `/etc/pihole/.ansible_password_hash`
  (re-runs `pihole setpassword` only when `pihole_web_password` changes).

Verify at `http://<pihole-ip>/admin`.

## Adding a new LXC

1. Add a host under `lxc_containers.hosts` in `inventory.yml` with `ctid`,
   `ansible_host`, `memory`, `disk`, `cores`.
2. Sync to PVE (see below).
3. `ansible-playbook playbooks/provision-lxc.yml` — creates and starts the CT
   with your SSH key already authorized.

## Adding a new service role

Use `roles/pihole/` as the template:

```
roles/<service>/
  defaults/main.yml         sane defaults
  tasks/main.yml            orchestrator (detect -> install -> configure)
  tasks/install.yml         install logic, gated by `creates:` for idempotency
  templates/...j2           config files
```

Add a playbook at `playbooks/<service>.yml` targeting the right host or group,
and put the host in that group in `inventory.yml`.

## Workstation → PVE sync

The repo lives on both the workstation (canonical) and the PVE node (where
Ansible runs). One-line sync:

```
rsync -av --delete \
  --exclude='.git' \
  --exclude='group_vars/all/secrets.yml' \
  ~/Documents/Personal/proxmox-homelab/ \
  root@<pve-ip>:/root/proxmox-homelab/
```

The excludes protect your gitignored secrets and local git history.

## Known limitations / not yet automated

- **Pi-hole DNS upstreams aren't reasserted post-install.** The role enforces
  the admin password but not the upstreams; those are set once via
  `setupVars.conf` at install time and managed by pi-hole itself afterward.
- **Pi-hole v5.x is not supported** — the role assumes v6+ (`pihole.toml`).
- **Nextcloud-AIO Ubuntu VM** is entirely manual.
- **No vault.** `secrets.yml` is plain YAML on disk. Encrypt with
  `ansible-vault encrypt group_vars/all/secrets.yml` before pushing the repo
  anywhere public.
- **Other services** (nginx-proxy, vaultwarden, uptime-kuma, homepage,
  stirling-pdf) have CTs but no roles — internal config is manual.
- **`community.general.proxmox` is deprecated** in favor of
  `community.proxmox`. The collection still works but a migration is on the
  near-term TODO.

## License

MIT — see `LICENSE`.

