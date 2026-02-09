# sharkshere-ansible

Ansible playbooks for configuring the sharkshere jump hosts (Caddy reverse proxy + Tailscale mesh VPN + SSH hardening).

## Architecture

```
Internet → jump-eu-central / jump-eu-north (Caddy, auto-TLS)
         → Tailscale tunnel
         → K8s services (via Tailscale operator)
```

Caddy's admin API listens on each host's Tailscale IP (`:2019`). A future K8s-side controller will push routes dynamically.

## Inventory

| Host | Location | IP |
|------|----------|----|
| jump-eu-central | Nuremberg (nbg1) | 78.47.240.158 |
| jump-eu-north | Helsinki (hel1) | 77.42.94.152 |

Hosts are provisioned by OpenTofu ([jumpingsharks](https://github.com/Yornik/jumpingsharks)).

## Roles

| Role | Description |
|------|-------------|
| **base** | apt upgrade, essential packages, timezone, unattended-upgrades |
| **ssh_hardening** | Disable password auth, limit auth tries, disable X11 forwarding |
| **fail2ban** | SSH brute-force protection |
| **tailscale** | Install and authenticate Tailscale for mesh connectivity |
| **caddy** | Install Caddy with admin API on Tailscale interface |

## Prerequisites

- Ansible (`pip install ansible`)
- SOPS + age key at `~/.config/sops/age/keys.txt`
- SSH key access to the jump hosts

## Usage

Decrypt secrets for the playbook run:

```bash
# Run against all jump hosts
ansible-playbook playbooks/site.yml

# Run against a single host
ansible-playbook playbooks/site.yml --limit jump-eu-central

# Dry run
ansible-playbook playbooks/site.yml --check --diff
```

## Secrets

Sensitive variables are SOPS-encrypted in `group_vars/jump_hosts/secrets.enc.yml`. Edit with:

```bash
sops group_vars/jump_hosts/secrets.enc.yml
```

## CI

Pull requests run three checks:
1. **yamllint** — YAML formatting
2. **ansible-lint** — Ansible best practices
3. **syntax-check** — `ansible-playbook --syntax-check`

All three checks must pass before merging to `main`.
