# VProfile Project: Monitoring & Observability Stack

This repository contains the provisioning scripts, application codebase, and configuration templates to deploy a robust, end-to-end monitoring and observability infrastructure. It leverages the **L-A-P-G** stack (**Loki**, **Alloy**, **Prometheus**, and **Grafana**) to collect, store, and visualize metrics and logs from a live web application environment.

---

## Architecture Overview

The monitoring architecture consists of two primary server types:
1. **Monitoring Node**: Runs central observability services (Prometheus, Loki, Grafana) to store and visualize data.
2. **Web Node (Target)**: Runs the application, collects local system metrics, and ships metrics/logs using Grafana Alloy.


## Deployment & Setup Guide

### Phase 1: Deploying the Monitoring Node

Run the setup scripts on your designated Monitoring Server (Ubuntu 22.04 LTS is recommended):

1.  **Install Prometheus:**
    ```bash
    chmod +x prometheus-setup.sh
    sudo ./prometheus-setup.sh
    ```
    *   Prometheus will start on port `9090`.
    *   Configuration file: `/etc/prometheus/prometheus.yml`

2.  **Install Loki:**
    ```bash
    chmod +x lokisetup.sh
    sudo ./lokisetup.sh
    ```
    *   Loki will start on port `3100`.
    *   Configuration file: `/etc/loki/config.yml`

3.  **Install Grafana Enterprise:**
    ```bash
    chmod +x grafana-setup.sh
    sudo ./grafana-setup.sh
    ```
    *   Access Grafana at `http://<Monitoring_IP>:3000` (Default credentials: `admin` / `admin`).

---

### Phase 2: Deploying the Web Node (App & Collectors)

On your target application server (Web Node):

1.  **Run the Web Node Setup Script:**
    Before running, update the script to match your Monitoring Server's IPs:
    *   Update `PrometheusIP` in the Alloy configuration block with your Prometheus server IP.
    *   Update `LokiIP` in the Alloy configuration block with your Loki server IP.

    ```bash
    chmod +x webnode_setup.sh
    sudo ./webnode_setup.sh
    ```
    This script automatically:
    *   Configures system settings and hostname (`web01`).
    *   Installs and runs **Prometheus Node Exporter** on port `9100`.
    *   Deploys the **Titan Flask App** on port `5000` and configures it under a systemd service.
    *   Installs and runs **Grafana Alloy** as the central telemetry agent.
    *   Launches background load generators to simulate CPU/disk stress and mock logs.

2.  **Configure Grafana Alloy:**
    If you need to manually modify the Grafana Alloy configuration, update `/etc/alloy/config.alloy`:
    ```hcl
    // Send metrics to Prometheus
    prometheus.remote_write "default" {
      endpoint {
        url = "http://<Prometheus_IP>:9090/api/v1/write"
      }
    }

    // Scrape local Flask metrics
    prometheus.scrape "metrics_5000" {
      targets = [{
        __address__ = "localhost:5000",
        __metrics_path__ = "/metrics",
      }]
      forward_to = [prometheus.remote_write.default.receiver]
    }

    // Capture logs and forward to Loki
    local.file_match "titan_logs" {
      path_targets = [{
        __path__ = "/var/log/titan/*.log",
        job      = "titan",
        hostname = constants.hostname,
      }]
    }

    loki.source.file "log_scrape" {
      targets    = local.file_match.titan_logs.targets
      forward_to = [loki.write.loki.receiver]
    }

    loki.write "loki" {
      endpoint {
        url = "http://<Loki_IP>:3100/loki/api/v1/push"
      }
    }
    ```
    Restart Alloy to apply changes:
    ```bash
    sudo systemctl restart alloy
    ```

---

### Phase 3: Traffic & Load Generation

To generate active traffic and simulate metrics fluctuations:

1.  **Generate HTTP requests:**
    Run the website test scripts by providing your Web Node IP:
    ```bash
    # Update MentionWebServerIPHere to your Web Node IP in the scripts, then run:
    ./WebsiteTest-main.sh &
    ./WebsiteTest-payment.sh &
    ```
2.  **Simulate Hardware Stress:**
    The script `load.sh` runs automatically in the background using `stress-ng` to generate randomized CPU spikes, memory usage, and direct disk write operations.

---

### Phase 4: Grafana Visualization

1.  Log into Grafana (`http://<Monitoring_IP>:3000`).
2.  Navigate to **Connections** > **Data Sources** > **Add data source**.
3.  Add **Prometheus**:
    *   URL: `http://localhost:9090`
4.  Add **Loki**:
    *   URL: `http://localhost:3100`
5.  Go to **Explore** to query your metrics (using PromQL) and application logs (using LogQL).
    *   Example Log Query: `{job="titan"}`
    *   Example Metric Query: `http_requests_total` or `node_cpu_seconds_total`
