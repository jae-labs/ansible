# Ansible

<p align="center">
  <a href="https://github.com/jae-labs/ansible/actions/workflows/ci.yml"><img src="https://github.com/jae-labs/ansible/actions/workflows/ci.yml/badge.svg?branch=main" alt="CI"></a>
  <a href="https://github.com/jae-labs/ansible/actions/workflows/ansible-adhoc.yml"><img src="https://github.com/jae-labs/ansible/actions/workflows/ansible-adhoc.yml/badge.svg?branch=main" alt="Ansible Ad-hoc Configuration"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/jae-labs/ansible" alt="License"></a>
  <a href="https://github.com/jae-labs/ansible/issues"><img src="https://img.shields.io/github/issues/jae-labs/ansible" alt="GitHub issues"></a>
  <a href="https://github.com/jae-labs/ansible/stargazers"><img src="https://img.shields.io/github/stars/jae-labs/ansible" alt="GitHub stars"></a>
  <a href="https://github.com/jae-labs/ansible/network"><img src="https://img.shields.io/github/forks/jae-labs/ansible" alt="GitHub forks"></a>
  <a href="https://docs.ansible.com/ansible/latest/roadmap/ROADMAP_2_21.html"><img src="https://img.shields.io/badge/ansible--core-%3E%3D%202.21-EE0000?logo=ansible&logoColor=white" alt="ansible-core >= 2.21"></a>
  <img src="https://img.shields.io/badge/platform-Ubuntu-E95420?logo=ubuntu&logoColor=white" alt="platform Ubuntu">
  <a href="https://buymeacoffee.com/luiz1361"><img src="https://img.shields.io/badge/Buy%20Me%20A%20Coffee-donate-orange.svg?logo=buymeacoffee" alt="Buy Me A Coffee"></a>
</p>

This repository is the source of truth for post-provision configuration of the OCI host managed by `jae-labs/terraform`. It applies baseline host hardening, service deployment, reverse proxying, and telemetry collection through a single inventory-backed playbook. Changes are organized by role and can be run locally or via the manual GitHub Actions workflow.

## What this repo manages

| Area | Purpose | Implementation |
|---|---|---|
| Inventory | Discover OCI instances and connect over SSH | `inventories/production/oci.oci.yml` |
| Baseline host config | OS updates, firewall, SSH hardening, fail2ban, swap, Docker, Nomad, OS hardening, auditd, Falco | `roles/common/`, `roles/iptables/`, `roles/ssh_hardening/`, `roles/fail2ban/`, `roles/swap/`, `roles/docker/`, `roles/nomad/`, `roles/os_hardening/`, `roles/auditd/`, `roles/falco/` |
| Web ingress | Traefik reverse proxy and Cloudflare Tunnel | `roles/traefik/`, `roles/cloudflare_tunnel/` |
| Observability | Grafana Alloy, cAdvisor, metrics, logs, traces export | `roles/grafana_alloy/`, `roles/cadvisor/` |

## How it works

- `playbooks/webservers.yml` applies all host configuration through role imports with stable role and category tags.
- OCI inventory is resolved through the `oracle.oci` inventory plugin.
- `bootstrap.sh` installs Python requirements into the same interpreter used by `ansible-playbook`, then installs required collections.
- Runtime secrets and service-specific settings are injected from local environment variables at deploy time.
- Operators can run the full playbook or limit execution by role/category tags.

Operational procedures, service-specific configuration, and observability details live in the `docs/` directory.

## Quick start

### Prerequisites

- Ansible installed locally
- OCI Python SDK installed through `requirements.txt`
- OCI credentials configured through the standard OCI config or environment flow
- SSH access to the target host with the matching private key available locally
- Public HTTPS hostname `oci-prod-1.justanother.engineer` routed through Cloudflare Tunnel to a Unix socket
- Tailscale SSH permission
- `TF_VAR_SSH_INGRESS_CIDR` **required** — the playbook will fail if it is empty or unset
- Required runtime environment variables for any services you deploy

Install dependencies:

```bash
bash bootstrap.sh
```

Install local Git hooks:

```bash
lefthook install
```

Inspect inventory:

```bash
ansible-inventory -i inventories/production/oci.oci.yml --list
```

Run the full playbook:

```bash
ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml
```

Run in check mode:

```bash
ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --check
```

## Validation

```bash
ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --syntax-check
ansible-inventory -i inventories/production/oci.oci.yml --list
ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --check
```

## Documentation

| Document | Description |
|---|---|
| [docs/operations.md](docs/operations.md) | Tag filtering, local execution patterns, and GitHub Actions ad-hoc runs |
| [docs/observability.md](docs/observability.md) | Grafana Alloy topology, scrape targets, and credential requirements |
