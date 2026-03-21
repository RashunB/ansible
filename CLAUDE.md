# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Write Lock

Do not write, edit, or commit any file in this repo without an explicit "implement" instruction from the user. Planning, exploring, and discussing are always safe.

---

## Behavior

- **Tone**: professional, precise, technically rigorous — no emojis
- **Default environment**: Ubuntu, bare-metal server, local Ansible execution
- **Repository standard**: GitHub
- **Before implementing**: validate ambiguous requests rather than acting on assumptions
- **Do not suggest** Docker-based rewrites — the systemd + binary deployment pattern is intentional
- **Do not suggest** skipping phases — the ordered progression (codify → validate destructively → test → tooling) is deliberate

---

## Common Commands

### Run the full stack
```
ansible-playbook playbooks/media_platform.yml
```

### Run a single service (tags match role names)
```
ansible-playbook playbooks/media_platform.yml --tags prowlarr
ansible-playbook playbooks/media_platform.yml --tags sonarr
ansible-playbook playbooks/media_platform.yml --tags radarr
ansible-playbook playbooks/media_platform.yml --tags sabnzbd
```

### Run a subset of the media_app pipeline for a specific service
```
ansible-playbook playbooks/media_platform.yml --tags "prowlarr,download"
ansible-playbook playbooks/media_platform.yml --tags "sonarr,service"
```
Available pipeline tags: `packages`, `users`, `dirs`, `download`, `link`, `pre_service`, `service`

### Bootstrap a new host (no Python dependency required)
```
ansible-playbook playbooks/bootstrap.yml
```

### Lint
```
ansible-lint
pre-commit run --all-files
```

### Install collections
```
ansible-galaxy collection install -r requirements.yml
```

---

## Architecture

### The `media_app` Role — Core Abstraction

All service roles (prowlarr, sonarr, radarr, sabnzbd) are thin wrappers. Each calls `include_role: name: media_app` with service-specific variable overrides. `media_app` executes a six-stage pipeline defined in `roles/media_app/tasks/main.yml`:

| Stage | Task File | Purpose |
|---|---|---|
| 1 | `packages.yml` | Install system packages |
| 2 | `users.yml` | Create shared `media` group + per-app service user |
| 3 | `dirs.yml` | Scaffold `/tank/appdata/{app}/` and `/srv/{app}/` trees |
| 4 | `download.yml` | Fetch release tarball, verify SHA256, decompress versioned archive |
| 5 | `link.yml` | Create/update symlink from `appdata/{app}/app` → versioned binary |
| 6 | `service.yml` | Render systemd unit via `templates/app_service.j2`, start service |

If `media_app_pre_service_hook` is defined, that task file runs between stages 5 and 6. SABnzbd uses this for Python venv creation (`roles/sabnzbd/tasks/python_venv.yml`).

### Path Structure

```
/tank/appdata/{app}/             # ZFS dataset root
  versions/                      # Extracted versioned releases
  app  ->  versions/{name}_{ver}/{archive_name}/
                                 # Active symlink — relink to roll back
  data/                          # Persistent app config

/srv/{app}/                      # Runtime working tree (bind-mounted into unit)
  app                            # Symlink mirroring appdata/{app}/app
  data/                          # App config (bind from appdata)
  downloads/                     # Where applicable (sonarr, radarr, sabnzbd)
```

### Adding a New App Role

1. Create `roles/{appname}/tasks/main.yml`
2. Call `include_role: name: media_app` — full parameter reference in `roles/media_app/defaults/main.yml`
3. Add the role to `playbooks/media_platform.yml` with a tag matching the role name

Mandatory variables per app:

```yaml
media_app_name:              # e.g. bazarr
media_app_exec_name:         # executable name inside archive (may differ from app name)
media_app_version:
media_app_download_url:
media_app_sha256:            # sha256:<hex>
media_app_archive_name:      # directory name inside the extracted tarball
media_app_owner:             # dedicated service user
media_app_service_exec:      # full ExecStart string
media_app_service_bind_ro:   # list of "src:dest" read-only bind mounts
media_app_service_bind_rw:   # list of "src:dest" read-write bind mounts
```

### systemd Hardening

`roles/media_app/templates/app_service.j2` enforces strict isolation on every unit:

- `ProtectSystem=strict`, `PrivateTmp`, `PrivateDevices`, `ProtectHome`
- `NoNewPrivileges`, `CapabilityBoundingSet=` (empty), `AmbientCapabilities=` (empty)
- `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`
- `SystemCallFilter` whitelist (`@system-service @basic-io @file-system @network-io @process @signal @memlock`); explicit denylist for `@mount @raw-io @reboot @swap @module @debug`
- `BindReadOnlyPaths` / `BindPaths` from `media_app_service_bind_ro` / `media_app_service_bind_rw`

**Consequence**: any path not listed in the bind mount variables will be inaccessible to the process at runtime. Always enumerate every path the process reads or writes.

### Service Dependency Chain

```
network-online.target
       └── prowlarr.service
       └── sabnzbd.service
              └── sonarr.service  (Wants: sabnzbd + prowlarr)
              └── radarr.service  (Wants: sabnzbd + prowlarr)
```

### SABnzbd — Python Source Deployment

SABnzbd differs from the *arr apps: it ships as a Python source tarball. The pre-service hook (`python_venv.yml`) installs `python3-virtualenv`, creates a venv inside the versioned archive directory, and installs `requirements.txt` via pip before the systemd unit is deployed.

```
ExecStart: /srv/sabnzbd/app/venv/bin/python /srv/sabnzbd/app/SABnzbd.py \
           --server 192.168.0.63:8082 \
           --config-file /srv/sabnzbd/data/config/config.ini \
           --disable-file-log --console
```

The PPA `ppa:jcfp/sab-addons` is added before the `media_app` include to supply `par2-turbo`.

### Known Portability Issue

`ansible.cfg` hardcodes all paths to `/tank/appdata/ansible/` (inventory, roles_path, collections_path, fact cache). Converting these to relative paths is a Phase 1 priority.

---

## Inventory

Single host (`media_platform`), `ansible_connection=local`. Ansible runs on the same machine it configures — no SSH involved. Host vars set `appdata=/tank/appdata` and `service_bind_ip=192.168.0.63`.

---

## Roadmap Phase Reference

| Phase | Focus | Status |
|---|---|---|
| 1 | Complete codification: Bazarr, Jellyfin, Recyclarr, ZFS role, `ansible.cfg` path fixes | Current |
| 2 | Destructive rebuild validation — full stack teardown and replay | Next |
| 3 | Molecule unit tests per role | Deferred |
| 4 | CI/CD, observability, Terraform, PostgreSQL migration, Ansible Vault | Post-validation |

No secrets are currently managed by Ansible. The *arr apps configure API keys through their web UIs post-deployment. Vault infrastructure is deferred until Phase 4 (PostgreSQL migration requires database credentials).
