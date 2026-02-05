---
title: Prometheus
description: How to install Prometheus for metrics storage and querying
tags:
    - docker
    - prometheus
    - metrics
---

# Setup Prometheus

Prometheus is a time-series database designed for metrics collection and alerting. It scrapes metrics from configured targets and stores them for querying via PromQL.

??? info "Prerequisites"

    Before installing Prometheus, ensure you have:

    - **Docker** and **Docker Compose** installed on your system
    - A **Grafana instance** for visualization (optional but recommended)

??? question "Why Prometheus?"

    Prometheus is the industry standard for metrics collection:

    - **Pull-based model**: Scrapes metrics from targets at configured intervals
    - **Remote write support**: Can receive metrics pushed from collectors like Alloy
    - **PromQL**: Powerful query language for analyzing time-series data
    - **Alerting**: Built-in alerting rules and Alertmanager integration
    - **Service discovery**: Automatically discover targets in dynamic environments

## Installation

Create a directory for Prometheus:

```bash
mkdir prometheus
```

Create the Docker Compose file:

```bash
nano prometheus/docker-compose.yml
```

```yaml title="docker-compose.yml"
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    environment:
      - TZ=Europe/Amsterdam # (1)!
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus:/prometheus # (2)!
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=90d' # (3)!
      - '--storage.tsdb.retention.size=100GB' # (4)!
      - '--web.enable-lifecycle' # (5)!
      - '--web.enable-remote-write-receiver' # (6)!
    restart: unless-stopped
    networks:
      - backend

networks:
  backend:
    name: backend

volumes:
  prometheus:
    name: prometheus
```

1. :material-clock-outline: **Timezone** - Change to your local timezone (e.g., `America/New_York`, `UTC`)
2. :material-database: **Data Volume** - Persistent storage for time-series data
3. :material-calendar-clock: **Time Retention** - Keep data for 90 days (whichever limit is hit first)
4. :material-harddisk: **Size Retention** - Maximum storage size before oldest data is deleted
5. :material-refresh: **Lifecycle API** - Allows reloading configuration without restart
6. :material-upload: **Remote Write Receiver** - Enables Alloy/agents to push metrics directly

## Configuration

Create the Prometheus configuration file:

```bash
nano prometheus/prometheus.yml
```

```yaml title="prometheus.yml"
global:
  scrape_interval: 15s # (1)!
  evaluation_interval: 15s # (2)!

scrape_configs: [] # (3)!
```

1. :material-timer: **Scrape Interval** - How often to collect metrics from targets
2. :material-calculator: **Evaluation Interval** - How often to evaluate alerting rules
3. :material-playlist-remove: **Scrape Configs** - Empty when using remote write from Alloy

??? tip "Remote Write vs Scraping"
    With `--web.enable-remote-write-receiver` enabled, collectors like Grafana Alloy can push metrics directly to Prometheus. This eliminates the need for scrape configurations and works better in distributed setups.

    If you need Prometheus to scrape targets directly, add them to `scrape_configs`:

    ```yaml
    scrape_configs:
      - job_name: 'example'
        static_configs:
          - targets: ['localhost:9090']
    ```

## Starting Prometheus

Start the Prometheus service:

```bash
docker compose -f prometheus/docker-compose.yml up -d
```

## Verification

After starting Prometheus, verify the installation:

**Check container status:**

```bash
docker ps | grep prometheus
```

**View Prometheus logs:**

```bash
docker logs prometheus
```

**Check Prometheus readiness:**

```bash
curl -s http://localhost:9090/-/ready
```

## Grafana Datasource

To visualize metrics in Grafana, add Prometheus as a datasource:

1. Open Grafana
2. Click **Connections** in the left-side menu
3. Search for **Prometheus**
4. Click **Add new Datasource**
5. Enter the name `prometheus`
6. Set the URL to `http://prometheus:9090`
7. Click **Save & Test**

[installation guide]: https://grafana.com/docs/prometheus/latest/setup/install/
