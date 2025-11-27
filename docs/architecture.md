# Observability Stack Architecture

This document outlines the architecture of the PLG OTEL Observability Stack, designed to run on a single LXC container via Docker Compose.

## 1. Component Overview

The stack consists of five primary components, each containerized and managed by Docker Compose:

*   **LXC Container**: The host environment for the entire stack. It provides the isolated Linux system where Docker is run. All stack ports are mapped to this container's IP address.
*   **Application**: Any external application running in a separate container or virtual machine. It is configured to send telemetry data (metrics, logs, and traces) to the OpenTelemetry Collector.
*   **OpenTelemetry Collector (otel-collector)**: The central data ingestion endpoint. It receives telemetry data in OTLP format (gRPC or HTTP), processes it, and routes it to the appropriate backends.
*   **Prometheus**: A time-series database that scrapes and stores metrics data from the OTel Collector. It is optimized for fast querying and alerting on numerical data.
*   **Loki**: A log aggregation system designed to store and query logs efficiently. It indexes metadata about logs rather than the full log content, making it cost-effective.
*   **Grafana Tempo**: A high-volume, distributed tracing backend. It stores and queries traces received from the OTel Collector.
*   **Grafana**: The visualization layer. It connects to Prometheus, Loki, and Tempo as data sources, allowing you to create dashboards, explore data, and set up alerts.

## 2. Data Flow

The data flows through the system in a unidirectional path, from your application to the storage backends, and is finally visualized in Grafana.

### Simple Diagrammatic Representation

```
+-----------------+      +-------------------------+      +-----------------+
|   Application   |----->|   OTel Collector        |----->|   Prometheus    |
| (Separate LXC)  |      |   (gRPC: 4317, HTTP: 4318)  |      |   (Metrics)     |
+-----------------+      |                         |      +-----------------+
                       |                         |              ^
                       |------------------------->|   +-----------------+
                       |                         |   |   Grafana Tempo |
                       |------------------------->|   |     (Traces)    |
                       |                         |   +-----------------+
                       +-----------+-------------+              ^
                                   |                            | (Scrapes)
                                   v                            |
                             +-----+------+                     |
                             |   Loki     |                     |
                             |  (Logs)    |                     |
                             +------------+                     |
                                                                |
+---------------------------------------------------------------+
|
v
+-----------------+
|     Grafana     |
| (Visualization) |
+-----------------+
```

### 3. OTel Collector Pipelines

The OTel Collector is configured with three distinct pipelines to handle different types of telemetry data.

#### a. Metrics Pipeline
1.  **Receiver**: The `otlp` receiver listens on ports `4317` (gRPC) and `4318` (HTTP) for incoming metrics data from your application.
2.  **Processor**: The `batch` processor groups the metrics into batches to optimize processing and reduce the number of outgoing requests.
3.  **Exporter**: The `prometheus` exporter exposes the processed metrics on an internal endpoint (`http://otel-collector:8889/metrics`). Prometheus is configured to scrape this endpoint periodically to ingest the metrics.
4.  **Exporter (Debug)**: The `debug` exporter logs the metrics to the console, which is useful for verifying data flow during development.

#### b. Logs Pipeline
1.  **Receiver**: The `otlp` receiver listens on the same ports (`4317` and `4318`) for incoming log data.
2.  **Processor**: The `batch` processor groups the logs into batches.
3.  **Exporter**: The `otlphttp/loki` exporter pushes the batched logs directly to Loki's ingestion endpoint (`http://loki:3100/loki/api/v1/push`).
4.  **Exporter (Debug)**: The `debug` exporter logs the data to the console for verification.

#### c. Traces Pipeline
1.  **Receiver**: The `otlp` receiver accepts trace data.
2.  **Processor**: The `batch` processor groups the traces.
3.  **Exporter**: The `otlp/tempo` exporter sends traces to Grafana Tempo for long-term storage.
4.  **Exporter (Debug)**: The `debug` exporter logs the trace data to the console for debugging purposes.
