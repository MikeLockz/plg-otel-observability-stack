# Deployment Guide: PLG OTEL Observability Stack

This guide provides step-by-step instructions for deploying the observability stack on a Proxmox LXC container and configuring a separate application to send telemetry data to it.

## 1. Prerequisites (with Proxmox Root Access)

This section provides detailed commands for setting up the required LXC container from a Proxmox root shell, using Debian 12 as the base.

### Step 1: Download Debian 12 Template

First, ensure your Proxmox host has the latest list of available templates and download the Debian 12 standard template.

```sh
# Update the list of available templates
pveam update

# Download the Debian 12 standard template
pveam download local debian-12-standard
```

### Step 2: Create and Configure the LXC Container

Run the following command to create the LXC container. **You must replace the placeholder values** to match your environment.

*   **`--hostname`**: A name for your container (e.g., `observability`).
*   **`--password`**: A secure password for the `root` user.
*   **`--storage`**: The name of your Proxmox storage pool (e.g., `local-lvm`).
*   **`--net0`**: Your network configuration.
    *   `ip=192.168.1.100/24`: **Replace with your desired static IP address** and CIDR mask.
    *   `gw=192.168.1.1`: **Replace with your network's gateway IP**.

```sh
# Create the LXC container (replace placeholders)
pct create 100 /var/lib/vz/template/cache/debian-12-standard.tar.zst \
  --hostname observability \
  --password YOUR_SECURE_PASSWORD \
  --memory 2048 \
  --cores 2 \
  --rootfs local-lvm:20 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.100/24,gw=192.168.1.1 \
  --onboot 1 \
  --features nesting=1
```
*   **`100`**: The VM ID for the new container. You can change this to any available ID.
*   **`--features nesting=1`**: This is **required** to allow Docker to run inside the LXC.

### Step 3: Start the Container and Install Docker

1.  **Start the container**:
    ```sh
    pct start 100
    ```

2.  **Install Docker and Docker Compose**:
    The following command executes the Docker installation script directly inside the newly created container.

    ```sh
    pct exec 100 -- bash -c " \
      apt-get update && \
      apt-get install -y curl && \
      curl -fsSL https://get.docker.com -o get-docker.sh && \
      sh get-docker.sh && \
      rm get-docker.sh \
    "
    ```

3.  **Verify the Installation**:
    Check that Docker and Docker Compose were installed correctly.

    ```sh
    # Check Docker version
    pct exec 100 -- docker --version

    # Check Docker Compose version
    pct exec 100 -- docker compose version
    ```

Your LXC container is now prepared and running with Docker and Docker Compose installed. You can now proceed to deploy the observability stack.


## 2. Deploying the Stack

1.  **Clone the Repository**:
    Log into your LXC container via SSH and clone this repository:
    ```sh
    git clone https://github.com/MikeLockz/plg-otel-observability-stack.git
    cd plg-otel-observability-stack
    ```

2.  **Start the Services**:
    Use Docker Compose to build and start all the services in detached mode:
    ```sh
    docker-compose up -d
    ```

3.  **Verify the Services**:
    Check that all containers are running correctly:
    ```sh
    docker-compose ps
    ```
    You should see `grafana`, `loki`, `prometheus`, and `otel-collector` with a status of `Up`.

## 3. Configuring Your Application to Send Telemetry

Your application, running in a separate LXC or VM, needs to be instrumented with an OpenTelemetry SDK. The key is to configure the SDK's exporter to point to the static IP address of your observability stack's LXC.

Let `<OBSERVABILITY_LXC_IP>` be the static IP address of the LXC running the stack.

*   **Endpoint for gRPC**: `http://<OBSERVABILITY_LXC_IP>:4317`
*   **Endpoint for HTTP/protobuf**: `http://<OBSERVABILITY_LXC_IP>:4318/v1/traces` (or `/v1/metrics`, `/v1/logs`)

Below are examples for different languages.

**Example: Python**
Set the following environment variables for your Python application:
```sh
export OTEL_EXPORTER_OTLP_ENDPOINT="http://<OBSERVABILITY_LXC_IP>:4317"
export OTEL_EXPORTER_OTLP_PROTOCOL="grpc"
```

**Example: Node.js**
Configure your OTLP exporter to use the correct URL:
```javascript
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const traceExporter = new OTLPTraceExporter({
  url: 'http://<OBSERVABILITY_LXC_IP>:4317',
});
```

**Example: Java**
Set the following Java system property when running your application:
```sh
-Dotel.exporter.otlp.endpoint=http://<OBSERVABILITY_LXC_IP>:4317
```

## 4. Initial Grafana Setup

1.  **Access Grafana**:
    Open your web browser and navigate to `http://<OBSERVABILITY_LXC_IP>:3000`.

2.  **Login**:
    Use the default credentials:
    *   **Username**: `admin`
    *   **Password**: `admin`
    You will be prompted to change your password upon first login.

3.  **Add Prometheus Data Source**:
    *   Go to **Connections** > **Data Sources** > **Add new data source**.
    *   Select **Prometheus**.
    *   Set the **Prometheus server URL** to: `http://prometheus:9090`.
    *   Click **Save & Test**. You should see a "Data source is working" message.

4.  **Add Loki Data Source**:
    *   Go to **Connections** > **Data Sources** > **Add new data source**.
    *   Select **Loki**.
    *   Set the **Loki server URL** to: `http://loki:3100`.
    *   Click **Save & Test**. You should see a "Data source connected" message.

Your observability stack is now ready. You can start creating dashboards in Grafana to visualize the metrics and logs being sent from your application.
