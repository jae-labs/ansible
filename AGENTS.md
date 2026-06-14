# AGENTS.md — jae-labs/ansible

## Overview

This repo configures the OCI host managed by `jae-labs/terraform`.

- inventory discovery uses `inventories/production/oci.oci.yml`
- main entrypoint is `playbooks/webservers.yml`
- behavior is split by role under `roles/`
- operational conventions live in `docs/`

## How to work in this repo

- prefer changing the smallest relevant role instead of editing unrelated tasks
- keep tasks idempotent; repeat runs must converge cleanly
- preserve role/tag structure in `playbooks/webservers.yml`
- prefer role defaults for configurable values instead of hardcoding
- update docs in the same PR when operator workflow or runtime requirements change

## Repo shape

- `playbooks/webservers.yml` orchestrates all imported roles
- `roles/` contains all host and service behavior: `auditd`, `cadvisor`, `cloudflare_tunnel`, `common`, `docker`, `fail2ban`, `falco`, `grafana_alloy`, `iptables`, `nomad`, `os_hardening`, `ssh_hardening`, `swap`, `traefik`
- `inventories/` contains OCI inventory and shared host vars
- `docs/operations.md` documents tag-based execution patterns

## Validation

- install dependencies: `bash bootstrap.sh`
- install hooks: `lefthook install`
- inspect inventory: `ansible-inventory -i inventories/production/oci.oci.yml --list`
- lint: `ansible-lint`
- syntax check: `ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --syntax-check`
- dry run: `ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --check`
- local hook gate: `lefthook run pre-commit --all-files`

## Constraints

- changes here must stay aligned with the OCI host assumptions created by `jae-labs/terraform`
- avoid broad playbook reshuffles unless the tag model or execution order truly needs to change
