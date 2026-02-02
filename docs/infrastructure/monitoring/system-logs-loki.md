---
title: System Syslog
description: In Part 2 of the System Monitoring series, discover how to configure log monitoring for your systems using Loki and Grafana Alloy, visualized with Grafana.
tags:
- docker
- grafana
- loki
- alloy
---

Monitoring isn't just about metrics—it's about ensuring application health. Centralized logging with Loki and Grafana provides deeper insights by visualizing and searching logs, helping you quickly identify and resolve issues.

## Key Components

1. Loki - Log aggregation system that stores and indexes logs efficiently for querying and analysis.
2. Grafana Alloy - Next-generation telemetry collector that ships logs from files to Loki with powerful processing capabilities.
3. Grafana - Provides visualization and search capabilities for logs stored in Loki.

!!! info "Prerequisites"
    This guide assumes you have Docker, Docker Compose, and Grafana already installed. If you need to set up Grafana, check out the [Host & Container Monitoring guide][grafana-setup].

## Setup Loki

To set up Loki, we need to create a folder to hold both the `docker-compose.yml` and the configuration file.

First, create the folder for Loki:
```bash
mkdir loki
```

Open a new `docker-compose.yml` file for editing:

```bash
nano loki/docker-compose.yml
```
Paste the following content into the file:
```yaml title="docker-compose.yml"
services:
  loki:
    image: grafana/loki
    container_name: loki
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam
    expose:
      - 3100
    volumes:
      - ./loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki:/tmp
    command: -config.file=/etc/loki/loki-config.yaml
    networks:
      - backend

networks:
  backend:
    name: backend

volumes:
    loki:
      name: loki
```

Loki requires a configuration file to define which services to scrape for metrics. Create the configuration file:
```bash
nano loki/loki-config.yaml
```
Paste the following content into the file:
```yaml title="loki-config.yml"
---
auth_enabled: false

server:
  http_listen_address: 0.0.0.0
  http_listen_port: 3100
  grpc_listen_port: 9095
  log_level: info

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: memberlist

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  query_timeout: 600s
  retention_period: "365d"
  ingestion_rate_mb: 4
  ingestion_burst_size_mb: 6
  max_streams_per_user: 10000
  max_line_size: 256000
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  creation_grace_period: 15m
  discover_log_levels: false

ruler:
  alertmanager_url: http://localhost:9093
```

**Configuration explained:**

- **auth_enabled**: Disabled for simplified single-user deployments (enable for multi-tenant setups)
- **server**: Configures HTTP (3100) and gRPC (9095) ports for receiving logs and queries
- **storage.filesystem**: Stores log chunks and rules locally on the filesystem
- **retention_period**: Keeps logs for 365 days before deletion
- **ingestion_rate_mb/ingestion_burst_size_mb**: Limits log ingestion rate to prevent overwhelming the system (4MB/s sustained, 6MB/s burst)
- **max_line_size**: Maximum size of a single log line (256KB)
- **reject_old_samples_max_age**: Rejects logs older than 168 hours (7 days) to maintain data consistency

## Setup Grafana Alloy

To finalize your logging setup with Loki, you'll need to configure Grafana Alloy to collect and ship logs to Loki.

??? question "Why Grafana Alloy?"

    Grafana Alloy is the next-generation telemetry collector that replaces Promtail. It supports multiple data formats including logs, metrics, traces, and profiles. For file-based log collection, Alloy provides:

    - More efficient log processing and transformation
    - Native support for multiple output destinations
    - Lower resource usage compared to Promtail
    - Unified configuration for all telemetry types
    - Active development and long-term support

Start by creating a folder to store the `docker-compose.yml` and Alloy configuration files:

```bash
mkdir alloy
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
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam
    ports:
      - "12345:12345"
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy:ro
      - /var/log:/var/log:ro
      - alloy-data:/var/lib/alloy/data
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - /etc/alloy/config.alloy
    networks:
      - backend

networks:
  backend:
    name: backend

volumes:
  alloy-data:
    name: alloy-data
```

Now, create the Alloy configuration file:

```bash
nano alloy/config.alloy
```

Paste the following configuration:

```hcl title="config.alloy"
// System auth.log collection
local.file_match "authlog" {
  path_targets = [{
    __path__ = "/var/log/auth.log",
  }]
}

loki.source.file "authlog" {
  targets    = local.file_match.authlog.targets
  forward_to = [loki.process.authlog.receiver]
}

loki.process "authlog" {
  stage.static_labels {
    values = {
      job = "authlog",
    }
  }

  forward_to = [loki.write.default.receiver]
}

// System syslog collection
local.file_match "syslog" {
  path_targets = [{
    __path__ = "/var/log/syslog",
  }]
}

loki.source.file "syslog" {
  targets    = local.file_match.syslog.targets
  forward_to = [loki.process.syslog.receiver]
}

loki.process "syslog" {
  stage.static_labels {
    values = {
      job = "syslog",
    }
  }

  forward_to = [loki.write.default.receiver]
}

// Loki write endpoint
loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

**Configuration explained:**

- **local.file_match**: Defines which log files to monitor (supports wildcards like `/var/log/*.log`)
- **loki.source.file**: Creates a file reader that tails log files and tracks reading positions
- **loki.process**: Processes logs and adds labels for organization in Grafana
- **stage.static_labels**: Adds custom labels to identify the log source (like `job: "syslog"`)
- **loki.write**: Defines the Loki endpoint where processed logs are sent

You can customize the file paths and labels according to your logging requirements. Alloy supports wildcards and multiple file sources.

Finally, start the Loki and Alloy services by running the following commands:

```bash
docker compose -f loki/docker-compose.yml up -d
docker compose -f alloy/docker-compose.yml up -d
```

## Grafana

To visualize logs from Loki in Grafana, you need to configure Loki as a datasource. Here’s how to do it:

1.	Open Grafana:
2.  Click **Connections** in the left-side menu.
3.  Search for **Loki**
4.  Click **Add new Datasource**
5.  Enter the name **loki**
6.  Fill in the Prometheus server URL `http://loki:3100`

### Exploring Logs in Grafana

Now that you have added Loki as a datasource, you can explore your logs:

1. In the left sidebar, click on Explore.
2. In the top-left dropdown menu, choose Loki as your datasource.
3. In the query section, select the label filename and set the value to /logs/syslog

## Summary

With Loki configured as a datasource in Grafana, Alloy will continuously collect and send log files to Loki, allowing you to visualize and analyze logs easily. This setup provides a comprehensive monitoring solution, enabling you to monitor both metrics and logs from your applications.

Grafana Alloy's modern architecture provides better performance and more flexible log processing compared to Promtail, while integrating seamlessly with the Grafana ecosystem.

[grafana-setup]: ./host-container-monitoring
