---
title: System Syslog
description: In Part 2 of the System Monitoring series, discover how to configure log monitoring for your systems using Loki and Promtail, visualized with Grafana.
tags:
- docker
- grafana
- loki
---

Monitoring isn’t just about metrics—it’s about ensuring application health. Centralized logging with Loki and Grafana provides deeper insights by visualizing and searching logs, helping you quickly identify and resolve issues.

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

## Setup Promtail

To finalize your logging setup with Loki, you’ll need to configure Promtail to send logs to Loki.

Start by creating a folder to store the `docker-compose.yml` and `promtail-config.yaml` files.

```bash
mkdir promtail
```

Open a new `docker-compose.yml` file for editing:

```bash
nano promtail/docker-compose.yml
```
Paste the following content into the file:
```yaml title="docker-compose.yml"
services:
  promtail:
    image: grafana/promtail
    container_name: promtail
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam
    volumes:
      - ./promtail-config.yaml:/etc/promtail/promtail-config.yaml:ro
      - /var/log/:/logs
    command: -config.file=/etc/promtail/promtail-config.yaml
    networks:
      - backend
networks:
  backend:
    name: backend
```

Now, create a configuration file named `promtail-config.yaml`:

```yaml title="promtail-config.yml"
server:
  http_listen_port: 9080
  grpc_listen_port: 0
positions:
  filename: /tmp/positions.yaml
clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
- job_name: authlog
  static_configs:
  - targets:
      - authlog
    labels:
      job: authlog
      __path__: /logs/auth.log

- job_name: syslog
  static_configs:
  - targets:
      - syslog
    labels:
      job: syslog
      __path__: /logs/syslog
```

This configuration will scrape the system’s auth and syslog logs.

Note: You can customize the job_name, `targets`, `job`, and `__path__` under scrape_configs according to your logging requirements.

Finally, start the Loki and Promtail services by running the following commands:

```bash
docker compose -f loki/docker-compose.yml up -d
docker compose -f promtail/docker-compose.yml up -d
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

With Loki configured as a datasource in Grafana, Promtail will continuously send log files to Loki, allowing you to visualize and analyze logs easily. This setup provides a comprehensive monitoring solution, enabling you to monitor both metrics and logs from your applications.
