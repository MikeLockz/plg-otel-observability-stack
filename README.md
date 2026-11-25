# PLG + OTel Observability Stack

A comprehensive, containerized observability stack combining **Prometheus**, **Grafana Mimir**, **Loki**, **Grafana**, **Tempo**, and the **OpenTelemetry Collector**. This stack provides a unified platform for metrics, logs, and traces, enabling full-stack observability for your applications and infrastructure with long-term storage capabilities.

## 🚀 Features

*   **Unified Visualization**: Grafana dashboards for metrics, logs, and traces.
*   **Centralized Logging**: Loki for efficient log aggregation and querying.
*   **Metrics Collection**: Prometheus for scraping and **Grafana Mimir** for long-term, scalable metrics storage.
*   **Distributed Tracing**: Tempo for high-volume trace storage and analysis.
*   **OpenTelemetry Support**: OTel Collector configured to ingest OTLP data (gRPC/HTTP).
*   **Infrastructure Monitoring**: Built-in collection of Host metrics (CPU, Memory, Disk) and Docker container stats.

## 🏗 Architecture

The stack runs entirely on Docker Compose. The **OpenTelemetry Collector** acts as the central agent, receiving telemetry from:
1.  **Applications** (via OTLP).
2.  **Host System** (via Host Metrics receiver).
3.  **Docker Engine** (via Docker Stats & Filelog receivers).

Data is then processed and routed to the respective backends:
*   **Metrics** → Prometheus → Mimir (Long-term Storage)
*   **Logs** → Loki
*   **Traces** → Tempo

For a detailed breakdown, see [Architecture Documentation](docs/architecture.md).

## 📋 Prerequisites

*   [Docker](https://docs.docker.com/get-docker/)
*   [Docker Compose](https://docs.docker.com/compose/install/)

## ⚡️ Quick Start

1.  **Clone the repository:**
    ```bash
    git clone <repository-url>
    cd plg-otel-observability-stack
    ```

2.  **Start the stack:**
    ```bash
    docker-compose up -d
    ```

3.  **Verify services are running:**
    ```bash
    docker-compose ps
    ```

## 🔌 Services & Ports

| Service | Port | Description |
| :--- | :--- | :--- |
| **Grafana** | `3000` | Visualization UI (User: `admin`, Pass: `admin`) |
| **Prometheus** | `9090` | Metrics Server |
| **Loki** | `3100` | Logs Aggregation System |
| **Tempo** | `3200` | Distributed Tracing Backend |
| **OTel Collector** | `4317` | OTLP gRPC Receiver |
| **OTel Collector** | `4318` | OTLP HTTP Receiver |
| **OTel Collector** | `8889` | Prometheus Exporter (Internal) |
| **Mimir** | `9009` | Long-term Metrics Storage |
| **Minio** | `9000` | S3-compatible Object Storage (UI) |

## ⚙️ Configuration

Configuration files are located in the `config/` directory:

*   `config/otel-collector.yml`: Configures receivers (Host, Docker, OTLP), processors, and exporters.
*   `config/prometheus.yml`: Prometheus scrape configs.
*   `config/loki-config.yml`: Loki storage and retention settings.
*   `config/tempo-config.yml`: Tempo configuration.
*   `config/mimir.yml`: Mimir monolithic configuration.
*   `config/mimir-alertmanager.yml`: Mimir Alertmanager configuration.
*   `config/grafana/`: Grafana provisioning for datasources and dashboards.

## 📊 Usage

### Accessing Dashboards
1.  Open your browser and navigate to `http://localhost:3000`.
2.  Login with default credentials:
    *   **Username:** `admin`
    *   **Password:** `admin`
3.  Explore the pre-provisioned dashboards or create your own.

### Sending Telemetry
Configure your applications to send OTLP data to the collector:
*   **gRPC Endpoint:** `localhost:4317`
*   **HTTP Endpoint:** `localhost:4318`

### Host & Docker Monitoring
The stack automatically starts monitoring the host machine and running Docker containers. You can query these metrics in Prometheus (e.g., `host_cpu_usage`, `docker_container_memory_usage`) or view them in Grafana.

## 📚 Documentation

*   [Architecture Overview](docs/architecture.md)
*   [Deployment Guide](docs/deployment_guide.md)

## Observability repair — 2026-11-24 (America/Denver)

LXC 900 now uses Aspen (`192.168.1.1`) for DNS, replacing stopped LXC 102.
The persistent Proxmox setting is owned by `lockdev-infra/proxmox/pve-configs/lxc/900.conf`.
For a running guest, update its PVE-managed `/etc/resolv.conf` as well; `pct set`
alone did not refresh that file. Restart existing Docker containers individually
to refresh Docker's cached upstream DNS, preserving their images and volumes.
Do not add a public fallback resolver: household names need Aspen's split DNS.

Pyroscope runs as UID/GID `10001:10001`. Its named volume
`plg-otel-observability-stack_pyroscope_data`, mounted at `/var/lib/pyroscope`,
was empty and root-owned (0755), preventing filesystem storage uploads and
cluster-seed writes. Corrected only the volume root ownership to `10001:10001`,
retaining 0755 and all volumes. For a fresh installation or restored volume,
inspect the running process UID/GID and Docker mount source first, then set that
volume root to the matching owner; never work around this by running Pyroscope
as root or granting world-write access. Existing ownership survives container
replacement. Restarting Pyroscope retries pending uploads.

The three failed Prometheus jobs (`el-dorado-server`,
`el-dorado-postgres-exporter`, `mcts-ai`) all target `192.168.1.44`, stopped
LXC 902. These are legacy deployment targets, unnecessary for monitoring the
current public Fly/Pages deployment. See `lockdev-docs/docs/tech/apps-host-plan.md`
for the decision to preserve 902 only as a recovery source. The three jobs were
removed from the versioned and live Prometheus configuration on 2026-11-24;
no target was repointed, no
legacy data deleted, and no stopped guest started. Keep any future public-app
monitoring separate from these obsolete LAN endpoints.

Verification: guest and Grafana-container DNS resolved `github.com` and
`home.lockdev.com` (household name returned `192.168.1.33`). After restart,
Pyroscope uploaded existing pending profile blocks to the named volume; its
storage upload success timestamp advanced, upload failure counter was zero,
and no permission errors appeared in post-restart logs. All nine containers
remain running; the two local Prometheus targets are up.

Prometheus target cleanup was validated with `promtool check config` (including
its five existing alert rules), then hot-reloaded with SIGHUP. The previous live
configuration is retained root-only at
`/root/maintenance/prometheus-target-cleanup-20261125T044123Z/prometheus.yml.before`
on LXC 900. Prometheus and OTel Collector are the only remaining active targets;
both report `up=1` and no scrape errors. Existing historical metrics, dashboards,
alert rules and remote write to Mimir are retained.
