# Proxmox Homelab — Ansible

Ansible-managed Proxmox single-node homelab. Replaces an earlier `init-all.sh`
bash script with idempotent playbooks for LXC provisioning and per-service
configuration.

## What this manages

- **LXC containers** — 6 CTs (nginx-proxy, pi-hole, vaultwarden, uptime-kuma,
  homepage, plus a `pi-hole-from-scratch` test CT). Creation,
  networking, root SSH key.
- **Pi-hole (v6+)** — installs the package when missing, then enforces the
  web admin password.
- **Vaultwarden** — installs Docker + the compose plugin, then runs
  `vaultwarden/server` from a pinned image with bind-mounted data.
- **Homepage** — installs Docker + the compose plugin, then runs
  `gethomepage/homepage` with a bind-mounted config dir.
- **Uptime Kuma** — installs Docker + the compose plugin, then runs
  `louislam/uptime-kuma` with a bind-mounted data dir.
- **nginx-proxy** — installs nginx (bare-metal, no Docker), then renders one
  reverse-proxy vhost per entry in `nginx_proxy_hosts` and takes full
  ownership of `sites-enabled/`. Also drops a small `proxy-headers.conf`
  http-context snippet that resolves real client IP from `CF-Connecting-IP`
  and trusts `X-Forwarded-Proto` from upstream proxies.
- **cloudflared** — installs Cloudflare's tunnel daemon on the same CT as
  nginx-proxy, templates an ingress rule per `nginx_proxy_hosts` entry
  routing each public hostname to local nginx, and ships a systemd unit.
  Single ingress, fans out by `Host:` header through nginx.
- **Docker engine** — shared `docker` role pulled in by any service role that
  needs it (vaultwarden, homepage, uptime-kuma). Ansible deduplicates the
  dependency within a play, so listing it from multiple roles is free.

The Nextcloud-AIO Ubuntu VM is still manual.

## Layout

```
ansible.cfg                          inventory + roles_path config
inventory.yml                        pve host + lxc_containers + service groups
group_vars/all/
  main.yml                           shared vars (template, gateway, ssh pubkey lookup)
  secrets.yml.example                template -> copy to secrets.yml (gitignored)
playbooks/
  provision-lxc.yml                  idempotent CT creation + start
  bootstrap-existing-lxc-keys.yml    one-time SSH key retrofit for pre-Ansible CTs
  configure-pihole.yml               install (if missing) + web admin password
  configure-vaultwarden.yml          install docker + run vaultwarden compose stack
  configure-homepage.yml             install docker + run homepage compose stack
  configure-uptime-kuma.yml          install docker + run uptime-kuma compose stack
  configure-nginx-proxy.yml          install nginx + render vhosts from inventory
  configure-cloudflared.yml          install cloudflared + render tunnel ingress
roles/docker/
  tasks/main.yml                     docker CE + compose plugin from upstream apt repo
roles/pihole/
  defaults/main.yml                  DNS upstreams, behavior toggles
  tasks/main.yml                     detect -> install -> password orchestrator
  tasks/install.yml                  apt deps, setupVars.conf preseed, unattended installer
  tasks/password.yml                 idempotent `pihole setpassword` via marker file
  templates/setupVars.conf.j2        install-time preseed (v6 migrates to pihole.toml)
roles/vaultwarden/
  defaults/main.yml                  pinned image, paths, port, DOMAIN, signups toggle
  meta/main.yml                      depends on `docker` role
  tasks/main.yml + tasks/deploy.yml  render compose -> `docker compose up -d`
  templates/docker-compose.yml.j2    single-service compose, env vars inline
roles/homepage/
  defaults/main.yml                  pinned image, paths, port, allowed-hosts
  meta/main.yml                      depends on `docker` role
  tasks/main.yml + tasks/deploy.yml  render compose -> `docker compose up -d`
  templates/docker-compose.yml.j2    single-service compose, /app/config bind-mount
  templates/{settings,services,bookmarks,widgets}.yaml.j2
                                     curated dashboard config rendered from inventory
roles/uptime-kuma/
  defaults/main.yml                  pinned image, paths, port
  meta/main.yml                      depends on `docker` role
  tasks/main.yml + tasks/deploy.yml  render compose -> `docker compose up -d`
  templates/docker-compose.yml.j2    single-service compose, /app/data bind-mount
roles/nginx-proxy/
  defaults/main.yml                  body size + timeouts, vhost schema, real-ip trust list
  tasks/main.yml + tasks/install.yml + tasks/configure.yml
                                     install nginx, render vhosts, prune stale
  templates/vhost.conf.j2            single template, ssl + backend_ssl knobs
  templates/proxy-headers.conf.j2    http-context map + realip module config
  handlers/main.yml                  reload nginx
roles/cloudflared/
  defaults/main.yml                  config dir + variable schema reference
  tasks/main.yml + tasks/install.yml + tasks/configure.yml
                                     apt repo, write credentials + config, ship unit
  templates/config.yml.j2            ingress rules templated from nginx_proxy_hosts
  templates/cloudflared.service.j2   systemd unit (we don't run `service install`)
  handlers/main.yml                  restart cloudflared
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

### Install + configure vaultwarden

```
ansible-playbook playbooks/configure-vaultwarden.yml
```

Runs against the `vaultwarden` group. The role pulls in `roles/docker/` via
its `meta/main.yml`, then renders `/opt/vaultwarden/docker-compose.yml` and
runs `docker compose up -d`. The rendered compose file embeds
`vaultwarden_admin_token` from `secrets.yml`; the file is mode `0600` and the
template task uses `no_log: true` so it never lands in ansible output.

First run flow: leave `vaultwarden_signups_allowed: "true"`, register the
first account at `http://<vaultwarden-ip>:8080/`, then flip the default to
`"false"` and re-run the playbook to disable open signups.

### Install + configure homepage

```
ansible-playbook playbooks/configure-homepage.yml
```

Runs against the `homepage` group. Same shape as vaultwarden — depends on
`roles/docker/`, renders compose, runs `docker compose up -d`. Reach the
dashboard at `http://<homepage-ip>:3000/`. The role also templates
`settings.yaml`, `services.yaml`, `bookmarks.yaml`, and `widgets.yaml` from
inventory; edit the templates under `roles/homepage/templates/`, not the
rendered files on the CT (Ansible owns them).

### Install + configure uptime-kuma

```
ansible-playbook playbooks/configure-uptime-kuma.yml
```

Runs against the `uptime-kuma` group. Same shape — depends on
`roles/docker/`, renders compose, runs `docker compose up -d`. Reach the
monitor at `http://<uptime-kuma-ip>:3001/` and complete the first-run
admin setup in the browser (uptime-kuma stores credentials in its sqlite
db, no env vars or secrets to template).

### Install + configure nginx-proxy

```
ansible-playbook playbooks/configure-nginx-proxy.yml
```

Runs against the `nginx-proxy` group. Installs nginx (bare-metal — no Docker
on this CT), renders one vhost per entry in `nginx_proxy_hosts` (set on the
`nginx-proxy` host in `inventory.yml`) into
`/etc/nginx/sites-available/<name>.conf`, symlinks each into
`sites-enabled/`, validates with `nginx -t`, and reloads on change. Anything
in `sites-enabled/` that isn't in the managed list — including Debian's
default `default` symlink — is removed.

Per-vhost schema (full reference lives in `roles/nginx-proxy/defaults/main.yml`):

- `name` / `domain` / `destination` — required.
- `ssl: true` (optional) — terminate TLS locally; needs `ssl_cert` +
  `ssl_key`. The cert files must already exist; this role does **not**
  manage certbot. Auto-generates a `:80 → :443` redirect.
- `backend_ssl: true` (optional) — set when the upstream is HTTPS with a
  self-signed cert (Proxmox 8006, Nextcloud-AIO admin :8080). Adds
  `proxy_ssl_verify off` + `proxy_ssl_server_name on`. The role infers this
  automatically when `destination` starts with `https://`, so the explicit
  flag is only needed for clarity.
- `redirect_root_to: /admin/` (optional) — 301 from `/` to a subpath. Used
  for Pi-hole, whose UI lives at `/admin/`.
- `extra_config` (optional) — raw nginx directives appended inside
  `location /` for one-off needs.

Defaults applied to every vhost: `client_max_body_size 10G` (Nextcloud
chunked uploads), websocket upgrade headers, `proxy_read_timeout 86400`
(24h — long enough for noVNC console + Vaultwarden live-sync).

Public hostnames are fronted by Cloudflare Tunnel; TLS is terminated at the
edge, so most entries can be HTTP-only here. The Nextcloud-AIO admin panel
is the exception — it self-signs at port 8080, so we terminate locally with
the existing letsencrypt cert and proxy upstream over HTTPS with verify off.

The role also drops `/etc/nginx/conf.d/proxy-headers.conf` — an http-context
snippet that (a) resolves real client IP from `CF-Connecting-IP` for any
request originating from a trusted proxy (default: loopback, i.e. cloudflared
on the same CT) and (b) preserves `X-Forwarded-Proto: https` when an upstream
proxy already terminated TLS. Override the trust list via
`nginx_real_ip_from` in inventory if you ever chain a different upstream.

### Install + configure cloudflared

```
ansible-playbook playbooks/configure-cloudflared.yml
```

Runs against the `cloudflared` group (currently just the nginx-proxy CT —
we co-locate cloudflared and nginx so every public hostname can fan out
through one local ingress). Adds Cloudflare's apt repo, installs the
`cloudflared` package, writes the tunnel credentials JSON to
`/etc/cloudflared/<UUID>.json` (mode 0600), templates `config.yml` with one
ingress rule per `nginx_proxy_hosts` entry, ships a systemd unit, runs
`cloudflared tunnel ingress validate` before enabling, then starts it.

**One-time setup before first run:**

1. From a workstation with a browser, run `cloudflared tunnel login` to mint
   `cert.pem`, then `cloudflared tunnel create <name>` to create the tunnel.
   That writes `~/.cloudflared/<UUID>.json`.
2. Copy the contents of the credentials JSON into
   `group_vars/all/secrets.yml` as a dict at `cloudflared_tunnel_credentials`
   (keys: `AccountTag`, `TunnelID`, `TunnelSecret`). `main.yml` reads
   `cloudflared_tunnel_id` out of the same dict, so the UUID lives in one
   place only.
3. For each public hostname, create a CNAME in Cloudflare DNS pointing
   `<hostname>.{{ cloudflare_zone }}` → `<UUID>.cfargotunnel.com`. Easiest
   from the workstation: `cloudflared tunnel route dns <name> <hostname>`
   per hostname. This role does **not** manage Cloudflare DNS.

**Ingress rule generation:**

- `ssl: false` vhost → `service: http://localhost:80` (default path).
- `ssl: true` vhost → `service: https://localhost:443` with
  `originServerName: <domain>` so SNI matches the local letsencrypt cert
  and TLS verification succeeds. Without this you'd get a redirect loop
  (cloudflared → nginx :80 → 301 → cloudflared → nginx :80 → …).
- Catch-all returns `http_status:404` for any tunnel hostname not in the
  list.

**Security note:** every entry in `nginx_proxy_hosts` becomes a tunnel
ingress rule. That means if Cloudflare DNS routes the hostname to your
tunnel, it's reachable from the public internet — including pi-hole admin
and the Proxmox web UI. If you want any of those LAN-only, either don't
create the CNAME for that hostname, or gate it with a Cloudflare Access
policy (Zero Trust dashboard → Access → Applications). This role doesn't
currently model "proxy via nginx but skip the tunnel" — extend
`config.yml.j2` with a per-host `tunnel: false` flag if you need it.
=======

### Docker-in-LXC: AppArmor override

Docker inside an unprivileged LXC trips over AppArmor at container init —
runc can't write `/proc/sys/net/ipv4/ip_unprivileged_port_start` and various
other operations are denied. The fix is one line in the CT's Proxmox
config:

```
lxc.apparmor.profile: unconfined
```

This is automated. Tag the host in `inventory.yml`:

```yaml
vaultwarden:
  ctid: 102
  ansible_host: 192.168.88.102
  ...
  lxc_docker_host: true
```

`provision-lxc.yml` has a second play that runs against the PVE node and
ensures the line is present in `/etc/pve/lxc/<ctid>.conf` for every host
flagged that way. If the line was newly added and the CT is running, the
play reboots it so the new profile takes effect.

Trade-off: AppArmor is fully disabled inside the CT, which reduces LXC
isolation. Acceptable for a homelab on a single node; not appropriate for
multi-tenant. The narrower workaround (`lxc.sysctl.net.ipv4.ip_unprivileged_port_start = 0`)
addresses only the one specific runc error and was the previous approach
here — superseded because new docker workloads kept tripping over
unrelated AppArmor denials.

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
- **Vaultwarden ADMIN_TOKEN is plaintext** in the rendered compose file. The
  upstream-recommended `argon2` hash form would be preferable; not done yet.
- **`community.general.proxmox` is deprecated** in favor of
  `community.proxmox`. The collection still works but a migration is on the
  near-term TODO.

## License

MIT — see `LICENSE`.

