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
ansible-playbook playbooks/media_platform.yml --tags bazarr
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

All service roles (prowlarr, sonarr, radarr, sabnzbd, bazarr) are thin wrappers. Each calls `include_role: name: media_app` with service-specific variable overrides. `media_app` executes a six-stage pipeline defined in `roles/media_app/tasks/main.yml`:

| Stage | Task File | Purpose |
|---|---|---|
| 1 | `packages.yml` | Install system packages |
| 2 | `users.yml` | Create shared `media` group + per-app service user + `arrservice` shared account |
| 3 | `dirs.yml` | Scaffold `/tank/appdata/{app}/` and `/srv/{app}/` trees |
| 4 | `download.yml` | Fetch release tarball, verify SHA256, decompress versioned archive |
| 5 | `link.yml` | Symlink `appdata/{app}/app` → versioned directory |
| 6 | `service.yml` | Render systemd unit via `templates/app_service.j2`, start service |

If `media_app_pre_service_hook` is defined, that task file runs between stages 5 and 6. SABnzbd and Bazarr use this for Python venv creation.

### Path Structure

```
/tank/appdata/{app}/
  versions/
    {app}_{ver}/                    # Extracted version root
      {ExecName}/                   # Binary apps: archive subdirectory (exec_root)
      {script}.py + venv/           # Python apps: script + venv at version root
  app  ->  versions/{app}_{ver}/   # Active symlink — relink to roll back
  data/                             # Persistent app config

/srv/{app}/                         # Runtime working tree (bind-mounted into unit)
  app                               # Bind target (exec_root or app, depending on app type)
  data/
  downloads/                        # Where applicable
```

**Key variable**: `media_app_exec_root = {{ media_app_root }}/app/{{ media_app_exec_name }}` — resolves through the `app` symlink to the exec directory inside the version folder. Binary apps use this as the bind mount source. Python apps with a nested archive (SABnzbd) override it explicitly; Python apps with a flat source layout (Bazarr) bind the `app` symlink directly.

### App-Specific Patterns

#### Binary *arr apps (Prowlarr, Sonarr, Radarr)
```
app → versions/{app}_{ver}/           # Symlink
app/{ExecName}/                       # exec_root — archive contents
app/{ExecName}/{ExecName}             # Actual binary
```
Bind: `exec_root` → `/srv/{app}/app` (read-only)
ExecStart: `/srv/{app}/app/{ExecName}/{ExecName} -nobrowser -data=/srv/{app}/data`

#### SABnzbd (Python source, nested archive)
```
app → versions/sabnzbd_4.5.3/        # Symlink
app/SABnzbd-4.5.3/                    # exec_root (overridden: archive_name subfolder)
app/SABnzbd-4.5.3/venv/              # Python venv (created by pre_service_hook)
```
Bind: `exec_root` → `/srv/sabnzbd/app` (read-only)
ExecStart: `/srv/sabnzbd/app/venv/bin/python /srv/sabnzbd/app/SABnzbd.py --server {ip}:8082 ...`
PPA `ppa:jcfp/sab-addons` added before `media_app` include to supply `par2-turbo`.

#### Bazarr (Python source, flat layout)
```
app → versions/bazarr_1.5.4/         # Symlink
app/bazarr.py                         # Script entry point
app/venv/                             # Python venv (created by pre_service_hook)
```
Bind: `app` → `/srv/bazarr/app` (read-write — Python needs write access for bytecode)
ExecStart: `/srv/bazarr/app/venv/bin/python /srv/bazarr/app/bazarr.py --config /srv/bazarr/data`

### Adding a New App Role

1. Create `roles/{appname}/tasks/main.yml`
2. Call `include_role: name: media_app` — full parameter reference in `roles/media_app/defaults/main.yml`
3. Add the role to `playbooks/media_platform.yml` with a tag matching the role name

Mandatory variables per app:

```yaml
media_app_name:              # e.g. jellyfin
media_app_exec_name:         # executable/archive name (defaults to capitalize(name))
media_app_version:
media_app_download_url:
media_app_sha256:            # sha256:<hex>
media_app_archive_name:      # directory name inside the extracted tarball (if differs from exec_name)
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
       └── bazarr.service
              └── sonarr.service  (Wants: sabnzbd + prowlarr)
              └── radarr.service  (Wants: sabnzbd + prowlarr)
```

Playbook role order: `common → prowlarr → sabnzbd → sonarr → radarr → bazarr`

### Known Portability Issue

`ansible.cfg` hardcodes all paths to `/tank/appdata/ansible/` (inventory, roles_path, collections_path, fact cache). Converting these to relative paths is a Phase 1 priority.

---

## Inventory

Single host (`media_platform`), `ansible_connection=local`. Ansible runs on the same machine it configures — no SSH involved. Host vars set `appdata=/tank/appdata` and `service_bind_ip=192.168.0.63`.

---

## Roadmap Phase Reference

| Phase | Focus | Status |
|---|---|---|
| 1 | Complete codification: Jellyfin, Recyclarr, ZFS role, `ansible.cfg` path fixes | Current |
| 2 | Destructive rebuild validation — full stack teardown and replay | Next |
| 3 | Molecule unit tests per role | Deferred |
| 4 | CI/CD, observability, Terraform, PostgreSQL migration, Ansible Vault | Post-validation |

**Working roles**: prowlarr, sonarr, radarr, sabnzbd, bazarr.
No secrets are currently managed by Ansible. API keys are configured through web UIs post-deployment. Vault infrastructure is deferred until Phase 4.
