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
