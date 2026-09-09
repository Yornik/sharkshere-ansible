# sharkshere-ansible

This repository configures the two edge hosts of the sharkshere platform with Ansible. It takes a new Debian 13 server from `jumpingsharks` and makes it a hardened TCP load balancer on the Tailscale mesh.

This document uses a style based on ASD-STE100 Simplified Technical English. Sentences are short. Each sentence gives one instruction or one fact.

## What this repository does

- It hardens SSH. Only keys can log in. Root cannot log in.
- It installs fail2ban for the host SSH service.
- It installs unattended-upgrades.
- It joins the host to the Tailscale mesh with an auth key from SOPS.
- It installs HAProxy. HAProxy forwards TCP on ports 80, 443 and 2222 into the cluster over Tailscale.

This repository is one of three:

| Repository | Layer | Function |
|---|---|---|
| [`jumpingsharks`](https://github.com/Yornik/jumpingsharks) | infrastructure | Creates the two Hetzner edge hosts and their DNS with OpenTofu. |
| `sharkshere-ansible` (this repository) | hosts | Hardens the edge hosts. Installs HAProxy, Tailscale and fail2ban. |
| [`sharkshere-gitops`](https://github.com/Yornik/sharkshere-gitops) | workloads | Reconciles the applications in the cluster with ArgoCD. |

## Documents

| Document | Content |
|---|---|
| [`docs/tech/README.md`](docs/tech/README.md) | Full technical overview: traffic flow diagram, HAProxy frontend table, role descriptions, design notes, constraints. |

## Repository layout

```text
ansible.cfg                          Ansible settings.
inventory/hosts.ini                  The two edge hosts. Generated from the jumpingsharks outputs.
inventory/group_vars/jump_hosts/     Variables. secrets.sops.yml is encrypted with SOPS.
playbooks/bootstrap.yml              First run as root. Creates the ansible user.
playbooks/site.yml                   Full configuration. Runs all roles in order.
roles/base                           Packages, timezone, unattended-upgrades.
roles/ssh_hardening                  sshd configuration.
roles/fail2ban                       fail2ban jail for sshd.
roles/tailscale                      Tailscale package and tailnet login.
roles/haproxy                        HAProxy package and haproxy.cfg.
```

## Before you start

Make sure that you have:

- Ansible 2.16 or later.
- The collections from `requirements.yml`. Install them with `ansible-galaxy collection install -r requirements.yml`.
- SOPS and the age key at `~/.config/sops/age/keys.txt`.
- SSH access to the hosts as `ansible`. For a new host, see [How to bootstrap a new host](#how-to-bootstrap-a-new-host).

## How to bootstrap a new host

Do this one time for each new host. It creates the `ansible` user.

1. Make sure that the host is in `inventory/hosts.ini`.
2. Run `ansible-playbook playbooks/bootstrap.yml -e ansible_user=root`.
3. Run the full configuration. See the next procedure.

## How to configure the hosts

1. Run `ansible-playbook playbooks/site.yml --check --diff`. Read the output.
2. Run `ansible-playbook playbooks/site.yml`.
3. To change one host only, add `--limit jump-eu-central` or `--limit jump-eu-north`.

Each role is idempotent. A second run makes no changes.

CAUTION: Do not edit `/etc/haproxy/haproxy.cfg` on the host. The next run replaces it. Edit `roles/haproxy/templates/haproxy.cfg.j2`.

## How to change the HAProxy configuration

1. Edit `roles/haproxy/templates/haproxy.cfg.j2`.
2. Run the configuration procedure above.

Ansible validates the new file with `haproxy -c` before it installs it. If the file has an error, the play stops. The old file stays in place. HAProxy stays up.

NOTE: The `:2222` frontend uses `init-addr last,libc,none`. With this setting, HAProxy starts even when the Tailscale name of the backend does not resolve yet.

## How to change a secret

1. Run `sops inventory/group_vars/jump_hosts/secrets.sops.yml`.
2. Edit the value. Save the file.
3. Commit the encrypted file.

CAUTION: Keep the `.sops.yml` file extension. The `community.sops` vars plugin reads only files with that extension.

## How to open a pull request

1. Make the change on a branch.
2. Open a pull request against `main`. CI runs three checks. See [CI checks](#ci-checks).
3. Merge the pull request.
4. Run the configuration procedure from `main`.

## CI checks

CI runs on each pull request. All three checks must pass before a merge.

| Check | What it does |
|---|---|
| `yamllint` | Checks the YAML files with the rules in `.yamllint`. |
| `ansible-lint` | Checks the playbooks and roles against Ansible best practice. |
| `ansible-playbook --syntax-check` | Checks the syntax of `playbooks/site.yml` and `playbooks/bootstrap.yml`. |

## Known limits

The edge has two hosts in two regions. The cluster behind it is in a home. The home has one power feed, one internet uplink and one NAS. These are accepted limits. See [`docs/tech/README.md`](docs/tech/README.md#homelab-constraints) for the reasons.
