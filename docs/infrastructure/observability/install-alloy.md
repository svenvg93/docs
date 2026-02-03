---
title: Grafana Alloy
description: How to install Grafana Alloy with a basic pluggable config
tags:
    - docker
    - alloy
    - grafana
---

# Setup Grafana Alloy

Grafana Alloy is a telemetry collector for logs, metrics, traces, and profiles. This guide covers installing Alloy with Docker to collect system logs and forward them to Loki.

??? info "Prerequisites"

    Before installing Alloy, ensure you have:

    - **Docker** and **Docker Compose** installed on your system
    - A running **Loki instance** to receive logs
    - Network connectivity between Alloy and your Loki endpoint
    - Read access to the log files you want to collect

??? question "Why Grafana Alloy?"

    Grafana Alloy is the next-generation telemetry collector that replaces Promtail. It supports multiple data formats including logs, metrics, traces, and profiles. For file-based log collection, Alloy provides:

    - More efficient log processing and transformation
    - Native support for multiple output destinations
    - Lower resource usage compared to Promtail
    - Unified configuration for all telemetry types
    - Active development and long-term support

## Installation

??? tip "Install on system level"
    This guide covers Docker installation. If you prefer to install Alloy directly on your system, follow the official [installation guide].

Start by creating a folder to store the `docker-compose.yml` and Alloy configuration files:

```bash
mkdir -p alloy/config
```

Create the Docker Compose file:

```bash
nano alloy/docker-compose.yml
```

Paste the following content:

```yaml title="docker-compose.yml"
services:
  alloy:
    image: grafana/alloy:latest
    container_name: alloy
    hostname: ${HOSTNAME} # (1)!
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam # (2)!
    ports:
      - "12345:12345" # (3)!
    volumes:
      - ./config/:/etc/alloy/config/:ro
      - /var/log:/var/log:ro # (4)!
      - alloy-data:/var/lib/alloy/data
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config/ # (5)!
    networks:
      - backend # (6)!

networks:
  backend:
    name: backend

volumes:
  alloy-data:
    name: alloy-data
```

1. :material-server: **Hostname** - Inherits the host system's hostname so logs are tagged with the correct source
2. :material-clock-outline: **Timezone** - Change to your local timezone (e.g., `America/New_York`, `UTC`)
3. :material-web: **Web UI Port** - Alloy's web interface for debugging and monitoring
4. :material-file-document-outline: **Log Directory** - Mount additional directories if your logs are elsewhere
5. :material-folder-cog: **Config Directory** - Alloy loads all `.alloy` files from this directory
6. :material-lan: **Network** - Ensure this matches the network where Loki is running

## Configuration Files

Alloy uses a modular configuration approach. We'll create separate config files for different purposes:

| File | Purpose |
|------|---------|
| `endpoint.alloy` | Centralized endpoints for Loki, Prometheus, etc. |
| `self.alloy` | Monitor Alloy's own metrics |

#### Endpoint Configuration

??? note "Config path for system install"
    When installed on system level, the default config directory is `/etc/alloy/`. Place your `.alloy` config files there instead of `./config/`.

    To use the modular config approach, edit `/etc/default/alloy` and change the `CONFIG_FILE` argument:

    ```bash
    # Change from single file:
    CONFIG_FILE="/etc/alloy/config.alloy"

    # To directory (loads all .alloy files):
    CONFIG_FILE="/etc/alloy/"
    ```

    Then restart the service: `sudo systemctl restart alloy`

Create the endpoint configuration file to centralize all your write endpoints:

```bash
nano alloy/config/endpoint.alloy
```

```hcl title="endpoint.alloy"
// Loki endpoint for log ingestion
loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push" // (1)!
  }
}

// Prometheus remote write endpoint for metrics
prometheus.remote_write "default" {
  endpoint {
    url = "http://prometheus:9090/api/v1/write" // (2)!
  }
}
```

1. :material-database-export: **Loki URL** - Change to your Loki endpoint. Use `loki` hostname if on the same Docker network, or a full URL like `http://192.168.1.100:3100/loki/api/v1/push`
2. :material-chart-line: **Prometheus/Mimir URL** - Change to your Prometheus remote write or Mimir endpoint

#### Self-Monitoring Configuration

Create the self-monitoring configuration to track Alloy's own health:

```bash
nano alloy/config/self.alloy
```

```hcl title="self.alloy"
// Export Alloy's internal metrics
prometheus.exporter.self "alloy_metrics" {}

// Scrape the exported metrics
prometheus.scrape "alloy_metrics" {
  targets         = prometheus.exporter.self.alloy_metrics.targets
  scrape_interval = "60s" // (1)!
  forward_to      = [prometheus.remote_write.default.receiver] // (2)!
}
```

1. :material-timer-outline: **Scrape Interval** - How often to collect Alloy's metrics
2. :material-arrow-right: **Forward To** - References the endpoint defined in `endpoint.alloy`

**Configuration explained:**

- **prometheus.exporter.self**: Exposes Alloy's internal metrics (memory usage, component health, etc.)
- **prometheus.scrape**: Periodically collects metrics from the exporter
- **loki.write**: Centralized Loki endpoint used by all log collection configs
- **prometheus.remote_write**: Centralized Prometheus endpoint used by all metric configs

??? tip "Adding more collectors"
    You can add additional `.alloy` files to the `config/` directory for specific use cases like:

    - `syslog.alloy` - System log collection
    - `docker.alloy` - Docker container logs
    - `node.alloy` - Node exporter metrics

## Starting Alloy

Start the Alloy service:

```bash
docker compose -f alloy/docker-compose.yml up -d
```

## Verification

After starting Alloy, verify the installation:

**Check container status:**

```bash
docker ps | grep alloy
```

**View Alloy logs:**

```bash
docker logs alloy
```

**Access the Web UI:**

Open `http://localhost:12345` in your browser to access the Alloy debug interface. Here you can:

- View active components and their status
- Check configuration graph and data flow
- Debug configuration issues

**Verify metrics in Prometheus/Mimir:**

Query for Alloy's self-metrics:

```promql
alloy_component_controller_running_components
```

If metrics appear, your setup is working correctly.

[installation guide]: https://grafana.com/docs/alloy/latest/set-up/install/linux
