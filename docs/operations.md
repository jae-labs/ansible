# Operations

## Tag filtering

Every imported role in `playbooks/webservers.yml` carries both a role tag and, where useful, a category tag. Use tags to limit change scope and reduce run time.

| Role | Tags | Purpose |
|---|---|---|
| `common` | `common`, `baseline` | OS updates, unattended-upgrades |
| `hardening` | `hardening`, `security`, `baseline` | Repo-owned host hardening role for SSH, PAM, sysctl, mounts, banners, dotfiles, and compatibility exceptions |
| `iptables` | `iptables`, `firewall`, `baseline` | Host-level firewall (SSH-only ingress, DROP default) |
| `fail2ban` | `fail2ban`, `security`, `baseline` | SSH daemon abuse protection |
| `ubuntu_pro` | `ubuntu_pro`, `security`, `baseline` | Ubuntu Pro attachment plus `esm-infra`, `esm-apps`, `livepatch`, `usg`, and Landscape SaaS registration |
| `auditd` | `auditd`, `security`, `baseline` | Linux audit daemon, immutable rules (-e 1), failure mode -f 2 |
| `falco` | `falco`, `security`, `baseline` | eBPF syscall monitoring |
| `swap` | `swap`, `baseline` | Swap file |
| `tuned` | `tuned`, `baseline` | TuneD host tuning for Ubuntu: `network-latency` profile, `tuned.service` enabled/started normally, dynamic tuning disabled |
| `docker` | `docker`, `baseline` | Docker CE with userns-remap, no docker group access |
| `cadvisor` | `cadvisor`, `monitoring` | Container metrics exporter (read-only Docker socket) |
| `nomad` | `nomad`, `baseline` | HashiCorp Nomad agent; privileged containers disabled by default |
| `traefik` | `traefik`, `web` | Traefik reverse proxy |
| `cloudflare_tunnel` | `cloudflare_tunnel`, `web` | `cloudflared` tunnel to Traefik, metrics on :49312 |
| `grafana_alloy` | `grafana_alloy`, `monitoring` | Telemetry collector (non-root, CAP_DAC_READ_SEARCH) |

Local examples:

```bash
ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --tags baseline
ansible-playbook -i inventories/production/oci.oci.yml playbooks/webservers.yml --skip-tags monitoring
doppler run -- ansible-playbook --inventory inventories/production/oci.oci.yml playbooks/webservers.yml --tags ubuntu_pro
```

## Ad-hoc runs via GitHub Actions

Use the manual **Ansible Ad-hoc Configuration** workflow for infrequent host or service changes that should run from CI instead of a local operator shell.

1. Open the repository **Actions** tab.
2. Select **Ansible Ad-hoc Configuration**.
3. Choose the branch.
4. Select a configuration category such as `baseline` or `monitoring`.
5. Optionally set custom `tags` or `skip-tags`.
6. Run the workflow.

## Notes

- SSH connects via the OCI instance Name tag (`display_name`), which matches the Tailscale hostname.
- Local `ubuntu_pro` runs require `UBUNTU_PRO_TOKEN`, `LANDSCAPE_ACCOUNT_NAME`, and `LANDSCAPE_REGISTRATION_KEY` in the controller environment. `doppler run -- ...` is the intended local path.
- Tailscale install, enrollment, IP forwarding, exit-node advertisement, and Tailscale SSH happen during Terraform/cloud-init bootstrap.
- Hardening is owned by the local `roles/hardening` role. It is implementation-only and intentionally does not include benchmark or audit/reporting scaffolding.
- `TF_VAR_SSH_INGRESS_CIDR` is **required**. The playbook will fail if it is empty or unset.
- Public HTTPS ingress is on `https://oci-prod-1.justanother.engineer` via Cloudflare Tunnel over a Unix socket. No ports 80 or 443 are open on the host firewall.
- Docker containers run under `userns-remap` (dockremap). No user has docker group access; container management is through systemd only.
- Alloy runs as the `alloy` user with `CAP_DAC_READ_SEARCH`, not as root.
- Auditd rules are locked immutable (`-e 1`) after load. Re-applying the auditd role unlocks, reloads, and re-locks via the handler sequence.
- Out of scope for this repository: `containerctl`, partitioning.
- Nomad docker driver: privileged containers are disabled by default (`nomad_docker_allow_privileged: false`). Opt in per-host only for workloads that genuinely require container-escape primitives (e.g. Docker-in-Docker, certain CSI/sysbox drivers). Enabling this setting allows any Nomad job to run privileged containers on the host.
