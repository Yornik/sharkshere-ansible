# sharkshere-ansible

Host configuration and hardening layer for the sharkshere public edge.

This repo is the operational bridge between cloud provisioning and cluster workloads:

1. `jumpingsharks`: creates the hosts and network surface
2. `sharkshere-ansible` (this repo): applies secure baseline and edge services
3. `sharkshere-gitops`: serves workloads behind the edge and mesh

## Managed Hosts

| Host | Location | Type | Role |
|------|----------|------|------|
| `jump-eu-central` | Nuremberg (`nbg1`) | `CX23` | edge reverse proxy + mesh endpoint |
| `jump-eu-north` | Helsinki (`hel1`) | `CX23` | edge reverse proxy + mesh endpoint |

Inventory is generated from OpenTofu outputs in `jumpingsharks`.

## Architecture

```text
Internet
  -> jump hosts (Caddy + TLS + host controls)
  -> Tailscale mesh
  -> private services / Kubernetes ingress
```

Caddy admin API is intentionally bound to Tailscale (`:2019`) instead of public interfaces.

## Roles

| Role | Responsibility |
|------|----------------|
| `base` | package updates, baseline packages, timezone, unattended upgrades |
| `ssh_hardening` | key-only auth and stricter SSH posture |
| `fail2ban` | brute-force protection |
| `tailscale` | private connectivity between edge and internal network |
| `caddy` | reverse proxy and cert automation |

## Design Choices

- Configuration is idempotent and role-scoped for predictable re-runs.
- Security defaults are part of baseline, not post-deploy checklist items.
- Public exposure is minimized; management paths prefer private mesh.
- CI validates syntax and lint before host changes are merged.

## Homelab Constraints

This edge layer improves exposure and security posture, but it does not remove core homelab constraints:

- Single power source at the home site
- Single residential ISP uplink
- Upstream dependency on shared NAS storage for some workloads

These are explicitly accepted risks. Solving them to near-datacenter standards is possible, but currently not practical for homelab economics and operational overhead.

## Prerequisites

- Ansible
- SSH key access to jump hosts
- SOPS + age key (`~/.config/sops/age/keys.txt`)

Secrets: `group_vars/jump_hosts/secrets.enc.yml`

## Usage

```bash
# all hosts
ansible-playbook playbooks/site.yml

# single host
ansible-playbook playbooks/site.yml --limit jump-eu-central

# dry run
ansible-playbook playbooks/site.yml --check --diff
```

## Secrets

```bash
sops group_vars/jump_hosts/secrets.enc.yml
```

## CI

PR checks:

1. `yamllint`
2. `ansible-lint`
3. `ansible-playbook --syntax-check`
