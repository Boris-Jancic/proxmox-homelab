# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment

Ansible runs **directly on the Proxmox host** (not a remote workstation). Always activate the venv before running any Ansible command:

```bash
source ~/ansible-env/bin/activate
```

Proxmox API is reached at `127.0.0.1:8006` (loopback). API auth uses **password** (`root@pam` + `proxmox_api_password` from `secrets.yml`) — token auth breaks LXC provisioning with local storage.

## Common commands

```bash
# Provision / start all LXC containers
ansible-playbook playbooks/1-provision-lxc.yml

# Provision Ubuntu VMs (downloads cloud image, creates VM, cloud-init, waits for SSH)
ansible-playbook playbooks/3-provision-vms.yml

# One-time SSH key retrofit for existing containers
ansible-playbook playbooks/2-bootstrap-existing-lxc-keys.yml

# Configure a single service
ansible-playbook playbooks/configure-<service>.yml

# Dry-run (works for configure plays, not provision-lxc)
ansible-playbook playbooks/configure-<service>.yml --check

# Run only specific tasks by tag
ansible-playbook playbooks/configure-<service>.yml --tags install

# Limit to one host
ansible-playbook playbooks/configure-nginx-proxy.yml --limit nginx-proxy
```

If `secrets.yml` is vault-encrypted, append `--ask-vault-pass` to any command above.

## Architecture

Two distinct play types — never mix them:

1. **Provision** (`1-provision-lxc.yml`) — talks to Proxmox API via `community.proxmox.proxmox` (localhost connection). Creates LXCs, injects SSH key, and writes `lxc.apparmor.profile: unconfined` into `/etc/pve/lxc/<ctid>.conf` for hosts tagged `lxc_docker_host: true`.

2. **Configure** (`configure-*.yml`) — SSHes into the target LXC or the PVE host, runs a single named role.

### Role shape

All roles follow `roles/pihole/` as the canonical example:
- `defaults/main.yml` — all tunables with defaults
- `tasks/main.yml` — thin orchestrator (`include_tasks`)
- `tasks/install.yml` — gated with `creates:` or `when:` so re-runs are cheap
- `templates/` — Jinja2 configs
- `handlers/main.yml` — service reload/restart

Roles that need Docker declare `meta/main.yml` → `dependencies: [role: docker]`. Ansible deduplicates per play, so Docker is installed exactly once even if multiple roles depend on it.

### Secrets & variables

- `group_vars/all/main.yml` — non-secret defaults (IPs, template VMID, `cloudflare_zone`)
- `group_vars/all/secrets.yml` — gitignored; copy from `secrets.yml.example` and fill in
- Per-host variables (e.g. `nginx_proxy_hosts`, `homepage_allowed_hosts`) live directly on the host entry in `inventory.yml`
- `cloudflare_zone` is defined once in `main.yml`; domains throughout `inventory.yml` reference it as `"subdomain.{{ cloudflare_zone }}"`

### Service map

| Host (CT) | IP | Runtime | Port |
|---|---|---|---|
| nginx-proxy (100) | .100 | bare nginx | 80/443 |
| pi-hole (101) | .101 | bare pihole | 80 |
| vaultwarden (102) | .102 | Docker compose | 8080 |
| uptime-kuma (103) | .103 | Docker compose | 3001 |
| homepage (104) | .104 | Docker compose | 3000 |
| ubuntu-vm1 (VM 110) | .110 | Docker compose (AIO) | 8080 (admin), 11000 (Nextcloud) |

## Known gotchas

- **`--check` fails for `1-provision-lxc.yml`** — `community.proxmox` skips itself in check mode. Run for real; existing CTs return `changed=0`.
- **New LXC may have empty `/etc/resolv.conf`** — set nameserver via Proxmox GUI before the configure play runs.
- **IPv6 hangs `apt`** — add `Acquire::ForceIPv4 "true"` to `/etc/apt/apt.conf.d/99force-ipv4` if apt stalls in a new CT.
- **`proxmoxer` + `requests` must be in the venv** — missing them causes cryptic import errors from the Proxmox module.
- **Never `pip install` into system Python on the PVE host** — Proxmox's own Python environment must not be modified.
- **nginx-proxy owns all of `sites-enabled/`** — any file there not in `nginx_proxy_hosts` (including Debian's `default`) is removed on each run.
- **Pi-hole password idempotency** keyed on marker file `/etc/pihole/.ansible_password_hash` — delete it to force a re-sync.
- **Vaultwarden**: first run needs `vaultwarden_signups_allowed: "true"` to register the first account, then flip to `"false"` and re-run.
- **Nextcloud AIO first run**: after `configure-nextcloud.yml`, visit `https://192.168.88.110:8080` to complete setup in the AIO admin UI. `NC_DOMAIN` is pre-filled from `nextcloud_aio_domain`. `SKIP_DOMAIN_VALIDATION` is set so the reverse-proxy setup doesn't block init.
- **docker role supports both Debian and Ubuntu** — repo URL uses `{{ ansible_distribution | lower }}` so the same role works for LXCs (Debian) and the Ubuntu VM.

## Adding a new service

1. Add a host entry under `lxc_containers.hosts` in `inventory.yml` (ctid, ansible_host, memory, disk, cores). Tag `lxc_docker_host: true` if it runs Docker.
2. Run `1-provision-lxc.yml` to create the CT.
3. Copy `roles/pihole/` shape; add `tasks/install.yml` gated with `creates:` for idempotency.
4. Add `playbooks/configure-<service>.yml` targeting the new inventory group.
5. Add a matching inventory group under `children:`.
6. Add an `nginx_proxy_hosts` entry on the `nginx-proxy` host in `inventory.yml`.
