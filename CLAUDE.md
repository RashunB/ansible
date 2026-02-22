# CLAUDE.md — Ansible Media Stack (Home Server)

## Project Location

```
# UPDATE THIS PATH BEFORE USE
~/projects/ansible-main (placeholder)
```

## What This Project Is

An Ansible-based Infrastructure as Code implementation for a media automation stack running on a home server. Critically: **Ansible is not the deployment tool — the stack already exists and runs.** Ansible is the codification and reproducibility layer built after the fact to:

- Enable full disaster recovery (rebuild from scratch in ~1 hour)
- Prevent configuration drift over time
- Document the system as code (playbooks describe desired state)
- Serve as a portfolio demonstration of IaC skills

This distinction matters for how the project is discussed and extended.

-----

## Current Ecosystem

### In Ansible (codified)

- **Prowlarr** — indexer management
- **Sonarr** — TV automation
- **Radarr** — movie automation

### Running but not yet in Ansible (codification gap)

- **SABnzbd** — Usenet download client
- **Bazarr** — subtitle management (integrates with Sonarr/Radarr)
- **Jellyfin** — media server
- **Recyclarr** — TRaSH Guides quality profile sync (adhoc, not scheduled)

### Infrastructure

- ZFS storage (`tank/appdata/` dataset structure)
- systemd service management with hardened unit files
- Ansible roles with Jinja2 templating

-----

## Architecture Notes

### Role Structure

The `media_app` role is the core abstraction. Each app role (prowlarr, sonarr, radarr) calls it with app-specific parameters rather than duplicating logic. This pattern makes adding new services straightforward and is the template for codifying the remaining services.

Task files follow a consistent split:

- `packages.yml` — system package installation
- `users.yml` — service user/group creation
- `dirs.yml` — directory scaffolding
- `download.yml` — binary/release fetching
- `link.yml` — symlink-based version management (blue-green style)
- `service.yml` — systemd unit deployment

### ZFS Integration

Data lives under `tank/appdata/`. Paths are currently hardcoded in `ansible.cfg` — converting to relative paths is an early priority to make the repo portable.

### Symlink Version Management

Releases are downloaded and versioned, with a symlink pointing to the active version. Enables rollbacks by relinking.

### Recyclarr

Currently running adhoc with TRaSH Guides defaults for 1080p movies and TV, with minor modifications to improve grab success. Configuration should be reviewed and understood before being codified — the existing tweaks need to be documented so intent is preserved when Ansible manages it.

-----

## Secrets

**There are no current secrets to manage.** The *arr apps configure API keys through their web UIs post-deployment. SABnzbd, Bazarr, and Jellyfin follow the same pattern. SQLite requires no credentials.

Ansible Vault infrastructure should be set up **ahead of the PostgreSQL migration (Phase 4)** when database credentials will need to be passed through Ansible. It is not a current gap.

-----

## Roadmap

The progression is deliberate and ordered. Each phase validates the previous before adding complexity.

### Phase 1 — Complete Codification (current focus)

Codify the remaining running services into Ansible roles following the existing `media_app` pattern:

1. SABnzbd role
1. Bazarr role
1. Jellyfin role
1. Recyclarr role (config management, not a service — different pattern)
1. Fix hardcoded paths in `ansible.cfg` (portability)
1. ZFS datasets role (declarative dataset + snapshot management)

**Exit criteria:** Every running service is described in Ansible. No manual steps required to configure a service post-deployment.

### Phase 2 — Destructive Validation (first acceptance test)

Tear down the entire stack. Run the playbooks. Walk away. Come back and open Jellyfin.

This is the real integration test. Gaps found here become commits that permanently eliminate manual steps. Each failure is expected and valuable — the goal is to find everything Ansible doesn't cover yet.

**Exit criteria:** Full stack rebuild from scratch with no manual intervention beyond running the playbook.

### Phase 3 — Molecule Unit Testing

Formalize what Phase 2 proved into repeatable, automated unit tests using Molecule. Each role gets tests that assert the correct end state.

**Exit criteria:** All roles have Molecule tests. A second destructive rebuild confirms both the playbooks and the tests are accurate.

### Phase 4 — DevOps Tooling (unordered, post-validation)

Once the foundation is proven, extend with:

- **CI/CD** — GitHub Actions for lint + Molecule on every push
- **Observability** — Prometheus + Grafana, informed by the question "what logs would have helped during Phase 2 troubleshooting"
- **Terraform** — infrastructure provisioning layer above Ansible
- **PostgreSQL migration** — *arr apps off SQLite, requires Ansible Vault for credentials
- **Ansible Vault** — secrets management ahead of PostgreSQL work

### Future State QoL (low priority, post-Phase 4)

- Readarr + Kavita (ebooks)
- Mylar3 + Komga (comics)
- Unpackerr (archive extraction gap-filler)
- Jellystat (Jellyfin playback analytics)
- Reverse proxy + Authelia (if external access ever needed)
- Jellyseerr (media requests, if sharing with others)

-----

## Observability Philosophy

Current observability is intentionally limited — log-based only, no metrics. This is a deferred decision, not a gap.

The approach for Phase 4: use Phase 2 troubleshooting experience to identify what questions couldn't be answered, then build observability to answer those specific questions. This produces intentional, justified monitoring rather than cargo-culted dashboards.

-----

## Assessment

|Area              |Rating|Notes                                                 |
|------------------|------|------------------------------------------------------|
|Architecture      |⭐⭐⭐⭐⭐ |Excellent role abstraction, media_app pattern is clean|
|Security hardening|⭐⭐⭐⭐⭐ |systemd hardening exceeds most production environments|
|Automation        |⭐⭐⭐⭐  |Strong Ansible fundamentals                           |
|Secrets management|N/A   |No current secrets — revisit at PostgreSQL migration  |
|Testing           |⭐⭐    |Lint only — Molecule is Phase 3                       |
|Observability     |⭐⭐    |Intentionally deferred, informed approach planned     |
|Documentation     |⭐⭐    |Minimal — improving via Obsidian vault                |

-----

## Context for AI Sessions

When working in this repo, assume:

- Skill level: strong Linux/Ansible/ZFS fundamentals, learning Molecule/CI patterns
- Goal: prove the stack is fully codified, then layer in testing and DevOps tooling methodically
- Career context: repositioning from IT Support → Infrastructure Automation / DevOps Engineer
- Learning style: understand the *why* before implementing, document takeaways in Obsidian
- Each completed phase produces both working infrastructure and documented understanding

**Do not suggest Docker-based rewrites.** The systemd + binary deployment pattern is intentional.

**Do not suggest skipping phases.** The ordered progression (codify → validate destructively → test → tooling) is deliberate.
