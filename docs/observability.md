# Observability

## Topology

Grafana Alloy runs as a non-root telemetry collector (`alloy` user, `CAP_DAC_READ_SEARCH`) on the OCI host. Each signal pipeline (metrics, logs, traces, profiles) is independently gated on its Grafana Cloud credentials. Missing credentials skip the entire pipeline so Alloy starts cleanly with a partial config.

| Signal | Endpoint | Notes |
|---|---|---|
| OTLP traces + metrics | `127.0.0.1:4317`, `127.0.0.1:4318` | gRPC and HTTP ingest; always on, outputs are conditional on credentials |
| Application metrics | `127.0.0.1:9090/metrics` | Application metrics scrape (bridge network, host loopback) |
| nginx metrics | `127.0.0.1:9113` | Exporter scrapes nginx stub_status on `127.0.0.1:8081` |
| cloudflared metrics | `127.0.0.1:49312` | Tunnel health and connection state |
| Alloy self-metrics | `127.0.0.1:12345/metrics` | Alloy self-observability |
| Container metrics | `127.0.0.1:8082/metrics` | cAdvisor, Docker container stats |
| Docker container logs | /var/run/docker.sock | Host-wide Docker container logs tailed by Alloy |
| System logs | journald | Relabeled dynamically into `example.service` |
| nginx logs | `/var/log/nginx/*.log` | Access + error logs (job="nginx") |
| Other log files | `/var/log/*.log` | `auth.log`, `kern.log`, `syslog`, `cloud-init`, `apt`, `dpkg`, `unattended-upgrades`, `fail2ban`, `falco`, `audit` |

Access logs are JSON and include `request_time`, `upstream_response_time`, `upstream_status`, and `upstream_addr` for latency and upstream error SLIs.

## Credential injection

Grafana Cloud write endpoints and tokens are injected from the local operator environment during deployment. Each signal is gated on its own triplet of credentials:

- **Metrics**: `GRAFANA_CLOUD_PROMETHEUS_URL`, `GRAFANA_CLOUD_PROMETHEUS_USERNAME`, `GRAFANA_CLOUD_PROMETHEUS_TOKEN`
- **Logs**: `GRAFANA_CLOUD_LOKI_URL`, `GRAFANA_CLOUD_LOKI_USERNAME`, `GRAFANA_CLOUD_LOKI_TOKEN`
- **Traces (Tempo)**: `GRAFANA_CLOUD_TEMPO_URL`, `GRAFANA_CLOUD_TEMPO_USERNAME`, `GRAFANA_CLOUD_TEMPO_TOKEN`
- **Profiles**: `GRAFANA_CLOUD_PROFILES_URL`, `GRAFANA_CLOUD_PROFILES_USERNAME`, `GRAFANA_CLOUD_PROFILES_TOKEN`

Additional required/secrets:

- `CLOUDFLARE_TUNNEL_TOKEN`
- `TF_VAR_SSH_INGRESS_CIDR`

Optional:

- `SENTRY_DSN`
- `SENTRY_ENVIRONMENT`
- `SENTRY_RELEASE`
- `SERVICE_VERSION`
- `CONCIERGE_TRACE_SAMPLE_PERCENTAGE` (default `1`)
- `CONCIERGE_TRACE_SLOW_THRESHOLD_MS` (default `1000`)

## Operational constraints

- If any credential triplet is missing, that signal pipeline is omitted entirely from the Alloy config. Alloy will start and operate with partial telemetry.
- The OTLP receiver is always on. If no Prometheus credentials are present, OTLP metrics are discarded. If no trace backends are configured, OTLP traces are discarded.
- Application traces are exported to Alloy first and forwarded to Tempo. Grafana Alloy performs stateful tail-based trace sampling on `conCIerge` traces (keeping all errors, all traces > 1s, and a 1% baseline sample of remaining traces). Docker container logs go to Loki through host-wide container log scraping; host logs continue through journald and file scraping.
- Loopback binding keeps all collector surfaces off the public interface; preserve that invariant.
- Alloy does not have Docker group access. Container metrics are collected by cAdvisor, which runs as a systemd service with read-only Docker socket access.

