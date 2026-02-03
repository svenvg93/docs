---
title: Grafana Loki
description: How to install Grafana Loki for log aggregation
tags:
    - docker
    - loki
    - grafana
---

# Setup Grafana Loki

Grafana Loki is a horizontally scalable, highly available log aggregation system. Unlike other logging systems, Loki indexes only metadata (labels) rather than the full text of log lines, making it cost-effective and efficient.

??? info "Prerequisites"

    Before installing Loki, ensure you have:

    - **Docker** and **Docker Compose** installed on your system
    - A **Grafana instance** for visualization
    - Sufficient disk space for log storage

??? question "Why Loki?"

    Loki is designed for efficiency and simplicity:

    - **Label-based indexing**: Only indexes metadata, not log content
    - **Cost-effective**: Requires less storage and compute than full-text indexing
    - **Grafana native**: Seamless integration with Grafana for querying and visualization
    - **LogQL**: Powerful query language similar to PromQL
    - **Multi-tenant**: Supports multiple tenants with data isolation

## Installation

Start by creating a folder for Loki:

```bash
mkdir loki
```

Create the Docker Compose file:

```bash
nano loki/docker-compose.yml
```

Paste the following content:

```yaml title="docker-compose.yml"
services:
  loki:
    image: grafana/loki
    container_name: loki
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam # (1)!
    expose:
      - 3100 # (2)!
    volumes:
      - ./loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki-data:/loki # (3)!
    command: -config.file=/etc/loki/loki-config.yaml
    networks:
      - backend # (4)!

networks:
  backend:
    name: backend

volumes:
  loki-data:
    name: loki-data
```

1. :material-clock-outline: **Timezone** - Change to your local timezone (e.g., `America/New_York`, `UTC`)
2. :material-network: **Port** - Exposes port 3100 internally on the Docker network (use `ports` to expose externally)
3. :material-database: **Data Volume** - Persistent storage for log chunks and indexes
4. :material-lan: **Network** - Ensure this matches the network where Alloy and Grafana are running

## Configuration

Create the Loki configuration file:

```bash
nano loki/loki-config.yaml
```

Paste the following configuration:

```yaml title="loki-config.yaml"
---
auth_enabled: false # (1)!

server:
  http_listen_address: 0.0.0.0
  http_listen_port: 3100 # (2)!
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
  retention_period: "365d" # (3)!
  ingestion_rate_mb: 4 # (4)!
  ingestion_burst_size_mb: 6
  max_streams_per_user: 10000
  max_line_size: 256000 # (5)!
  reject_old_samples: true
  reject_old_samples_max_age: 168h # (6)!
  creation_grace_period: 15m
  discover_log_levels: false

ruler:
  alertmanager_url: http://localhost:9093
```

1. :material-shield-off: **Auth** - Disabled for single-user deployments. Enable for multi-tenant setups
2. :material-web: **HTTP Port** - Port for receiving logs and queries via HTTP API
3. :material-calendar-clock: **Retention** - How long to keep logs (365 days). Adjust based on storage capacity
4. :material-speedometer: **Ingestion Rate** - Max ingestion rate in MB/s (4MB/s sustained, 6MB/s burst)
5. :material-text-long: **Max Line Size** - Maximum size of a single log line (256KB)
6. :material-clock-alert: **Reject Old Samples** - Rejects logs older than 7 days to maintain data consistency

**Configuration explained:**

- **auth_enabled**: Disabled for simplified single-user deployments (enable for multi-tenant setups)
- **server**: Configures HTTP (3100) and gRPC (9095) ports for receiving logs and queries
- **storage.filesystem**: Stores log chunks and rules locally on the filesystem
- **schema_config**: Defines how logs are indexed and stored (TSDB with 24h index periods)
- **limits_config**: Controls ingestion rates, retention, and query limits

## Starting Loki

Start the Loki service:

```bash
docker compose -f loki/docker-compose.yml up -d
```

## Verification

After starting Loki, verify the installation:

**Check container status:**

```bash
docker ps | grep loki
```

**View Loki logs:**

```bash
docker logs loki
```

**Check Loki readiness:**

```bash
curl -s http://localhost:3100/ready
```

Should return `ready` when Loki is fully initialized.

**Check Loki metrics:**

```bash
curl -s http://localhost:3100/metrics | head -20
```

## Grafana Datasource

To visualize logs in Grafana, add Loki as a datasource:

1. Open Grafana
2. Click **Connections** in the left-side menu
3. Search for **Loki**
4. Click **Add new Datasource**
5. Enter the name `loki`
6. Set the URL to `http://loki:3100`
7. Click **Save & Test**

??? tip "Query logs with LogQL"
    Once configured, you can query logs in Grafana Explore using LogQL:

    ```logql
    {job="syslog"} |= "error"
    ```

    This queries all logs with the label `job="syslog"` containing the word "error".

[installation guide]: https://grafana.com/docs/loki/latest/setup/install/
