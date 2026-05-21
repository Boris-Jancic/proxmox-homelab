# Proxmox Homelab — Ansible

Ansible-managed jerry-rigged single-node(for now) Proxmox homelab.
Replaces an earlier `init-all.sh` script which provisioned my LXC's.
As I continued to add more & more services the script balooned so I decided to make this repository to have everything managed as IaC.

## Services

| Role | CT | Port | Notes |
|---|---|---|---|
| pi-hole | pi-hole | 80 | v6+ only. Installs if missing, enforces web admin password. |
| vaultwarden | vaultwarden | 8080 | Docker compose. `ADMIN_TOKEN` from `secrets.yml`. |
| homepage | homepage | 3000 | Docker compose. Templates `settings/services/bookmarks/widgets.yaml` from inventory. |
| uptime-kuma | uptime-kuma | 3001 | Docker compose. First-run admin setup is browser-only. |
| pve-scripts-local | pve-scripts-local | 3000 | Bare-metal Node 22 + systemd. Clones [community-scripts/ProxmoxVE-Local](https://github.com/community-scripts/ProxmoxVE-Local), runs `npm run build`, served via `npm start`. |
| nextcloud-aio | ubuntu-vm1 (VM 110) | 8080 (admin), 11000 (Nextcloud) | Docker compose. Ubuntu 24.04 VM. AIO mastercontainer spawns all sub-services. Passphrase printed by Ansible after deploy. |
| nginx-proxy | nginx-proxy | 80/443 | Bare-metal nginx. One vhost per `nginx_proxy_hosts` entry. Owns `sites-enabled/`. |

Service roles needing Docker pull `roles/docker/` via `meta/main.yml`
(Ansible deduplicates per play). The `docker` role supports both Debian
(LXCs) and Ubuntu (VMs) via `ansible_distribution | lower`.

## Layout

```
ansible.cfg, inventory.yml, requirements.yml
group_vars/all/{main.yml, secrets.yml.example}
playbooks/                  one per service + provision-lxc + bootstrap-existing-lxc-keys
roles/<service>/            defaults, tasks, templates, handlers
```

## Prerequisites

- `pip install --user ansible proxmoxer requests`
- `ansible-galaxy collection install -r requirements.yml`
- ed25519 key at `~/.ssh/id_ed25519.pub` (override `controller_ssh_pubkey_path`
  in `group_vars/all/main.yml` for rsa).
- Root SSH to the PVE node.

## Initial setup

1. Set `pve.ansible_host` and adjust LXC CTIDs / IPs in `inventory.yml`.
2. `cp group_vars/all/secrets.yml.example group_vars/all/secrets.yml` and fill in.
3. `ssh-copy-id root@<pve-ip>`.

## Workflows

```
ansible-playbook playbooks/1-provision-lxc.yml               # create + start LXCs
ansible-playbook playbooks/2-bootstrap-existing-lxc-keys.yml # one-time SSH retrofit
ansible-playbook playbooks/3-provision-vms.yml               # create Ubuntu VM + base config
ansible-playbook playbooks/configure-<service>.yml           # per-service install + configure
ansible-playbook playbooks/configure-nextcloud.yml           # Nextcloud AIO (prints passphrase)
```

`--check` doesn't work for `provision-lxc` — `community.general.proxmox`
skips itself in check mode. Run for real; existing CTs report `changed=0`.

### Service notes

- **pi-hole** — first run is slow (~5–10 min). Password idempotency keyed on
  marker file `/etc/pihole/.ansible_password_hash`.
- **vaultwarden** — first run, leave `vaultwarden_signups_allowed: "true"`,
  register the first account at `http://<ip>:8080/`, then flip to `"false"`
  and re-run.
- **homepage** — Ansible owns `settings.yaml` etc. Edit the templates under
  `roles/homepage/templates/`, not the rendered files on the CT.
- **uptime-kuma** — credentials live in sqlite; no env vars to template.
- **nextcloud-aio** — Ubuntu VM (110) provisioned via `3-provision-vms.yml`. After
  `configure-nextcloud.yml`, Ansible prints the AIO admin passphrase. Visit
  `https://192.168.88.110:8080` to complete setup. `NC_DOMAIN` pre-filled from
  `nextcloud_aio_domain`. `AIO_PASSWORD` sets passphrase from `secrets.yml`.

### nginx-proxy vhost schema

Set `nginx_proxy_hosts` on the `nginx-proxy` host in `inventory.yml`. Per
entry:

- `name`, `domain`, `destination` — required.
- `ssl: true` — terminate TLS locally; needs `ssl_cert` + `ssl_key`. Cert
  files must already exist (this role does **not** run certbot).
  Auto-generates `:80 → :443` redirect.
- `backend_ssl: true` — `proxy_ssl_verify off` + `proxy_ssl_server_name on`
  for self-signed upstreams (Proxmox 8006, Nextcloud-AIO admin :8080).
  Inferred automatically from `https://` in `destination`.
- `redirect_root_to: /admin/` — 301 from `/` to subpath (used for pi-hole).
- `extra_config` — raw nginx directives appended inside `location /`.

Defaults: `client_max_body_size 10G`, websocket upgrade headers,
`proxy_read_timeout 86400`. Anything in `sites-enabled/` not in the managed
list (including Debian's `default`) is removed.

## Docker-in-LXC: AppArmor override

Docker in an unprivileged LXC trips AppArmor at runc init. Fix:

```
lxc.apparmor.profile: unconfined
```

Automated — tag the host in `inventory.yml` with `lxc_docker_host: true`.
`1-provision-lxc.yml`'s second play writes the line into
`/etc/pve/lxc/<ctid>.conf` and reboots the CT if newly added.

Trade-off: AppArmor fully disabled inside the CT. Fine for single-node
homelab, not multi-tenant. The narrower
`lxc.sysctl.net.ipv4.ip_unprivileged_port_start = 0` was the previous
approach — replaced because newer Docker workloads kept hitting unrelated
AppArmor denials.

## Adding things

**New LXC:** add under `lxc_containers.hosts` with `ctid`, `ansible_host`,
`memory`, `disk`, `cores`. Sync to PVE, then run `1-provision-lxc.yml`.

**New service role:** copy `roles/pihole/` shape (defaults, tasks/main.yml
orchestrator, tasks/install.yml gated with `creates:`, templates). Add a
playbook in `playbooks/`, add the host to the right inventory group.

## Workstation → PVE sync

```
rsync -av --delete \
  --exclude='.git' \
  --exclude='group_vars/all/secrets.yml' \
  ~/Documents/Personal/proxmox-homelab/ \
  root@<pve-ip>:/root/proxmox-homelab/
```

## Known limitations

- Pi-hole DNS upstreams set once via `setupVars.conf` at install, not reasserted.
- Pi-hole v5.x unsupported (assumes `pihole.toml`).
- Nextcloud-AIO first-run domain/storage setup is browser-only after Ansible deploy.
- No vault — `secrets.yml` is plain YAML. `ansible-vault encrypt` before
  pushing anywhere public.
- Vaultwarden `ADMIN_TOKEN` is plaintext in the rendered compose file
  (argon2 hash form preferred; not done).
- `community.general.proxmox` is deprecated in favor of `community.proxmox`;
  migration on the TODO.
