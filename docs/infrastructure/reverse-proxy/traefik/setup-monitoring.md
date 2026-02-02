---
title: Monitoring Setup
description: Learn how to monitor your Traefik instance's performance using Prometheus, Loki, and Grafana.
draft: false
tags:
- traefik
- metrics
- logging
---

Traefik exposes metrics on EntryPoints, Routers, Services, and more. This guide shows you how to collect these metrics with Prometheus and aggregate logs with Loki for a complete monitoring solution.

## Key Components

1. Traefik - Exposes metrics for monitoring various components of your services.
2. Prometheus - Scrapes and stores metrics data from Traefik, providing a time-series database for easy access.
3. Loki - Aggregates log data, allowing for centralized logging alongside metrics.
4. Grafana Alloy - Ships logs from your applications to Loki for efficient storage and querying.

!!! info "Prerequisites"
    This guide assumes you already have Grafana, Prometheus, Alloy, and Loki set up. See the [Host & Container Monitoring][grafana-prometheus] and [System Logs with Loki][alloy-loki] guides for setup instructions.

## Metrics

### Traefik

To enable metrics in Traefik, add the following command flags to your `docker-compose.yml`:

```yaml title="docker-compose.yml"
command:
  # ... existing commands ...
  - "--metrics.prometheus=true"
  - "--metrics.prometheus.addEntryPointsLabels=true"
  - "--metrics.prometheus.addRoutersLabels=true"
  - "--metrics.prometheus.addServicesLabels=true"
```

By default, Traefik exposes metrics at `:8080/metrics`.

Re-create the Traefik container to apply the changes:

```bash
docker compose -f traefik/docker-compose.yml up -d --force-recreate
```

Verify metrics are available by visiting `http://<traefik-ip>:8080/metrics` (replace `<traefik-ip>` with your Traefik server's IP).

??? tip "Accessing Metrics Externally"
    If you need to access metrics from outside the Docker network (e.g., from another machine), add port 8080 to the ports section in your `docker-compose.yml`:

    ```yaml
    ports:
      - 80:80
      - 443:443
      - 8080:8080  # For external metrics access
    ```

### Prometheus
To ensure Prometheus collects metrics from Traefik, you need to add a scrape configuration to your existing prometheus configuration.

Add the following configuration to your `prometheus.yml`:
```yaml title="prometheus.yml"
scrape_configs:
  - job_name: 'traefik'
    scrape_interval: 5s
    static_configs:
      - targets: ['<traefik-ip>:8080']
```
Replace `<traefik-ip>` with your Traefik server's IP address.

To apply the configuration changes, restart the Prometheus containers:

```bash
docker restart prometheus
```

After the container is restarted, Prometheus will start scraping metrics from Traefik at the defined interval. You can now visualize these metrics in Grafana or any other monitoring tool you are using.

## Log files

To effectively monitor logs in Traefik, you’ll need to configure both Traefik logs and access logs. Follow these steps to set everything up.

### Traefik

Add logging configuration to your `docker-compose.yml`:

1. **Add command flags** for logging:

```yaml title="docker-compose.yml"
command:
  # ... existing commands ...
  - "--accesslog=true"
  - "--accesslog.filepath=/log/access.log"
  - "--accesslog.format=json"
  - "--accesslog.fields.defaultmode=keep"
  - "--accesslog.fields.names.StartUTC=drop"
  - "--log.filepath=/log/traefik.log"
  - "--log.format=json"
```

2. **Add volume mapping** for log directory:

```yaml title="docker-compose.yml"
volumes:
  # ... existing volumes ...
  - /var/log/traefik:/log
```

**Configuration explained:**

- **Access Logs**: Logs every request (client IP, HTTP status, method, etc.) in JSON format
- **Traefik Logs**: Captures Traefik service events (startup, shutdown, config changes) in JSON format

Re-create the Traefik container to apply the changes:

```bash
docker compose -f traefik/docker-compose.yml up -d --force-recreate
```

### Grafana Alloy

To enable Alloy to collect Traefik logs, you'll need to add a new file source configuration to your `config.alloy`. Add the following to your existing Alloy configuration:

```hcl title="config.alloy"
// Traefik logs collection
local.file_match "traefik" {
  path_targets = [{
    __path__ = "/var/log/traefik/*.log",
  }]
}

loki.source.file "traefik" {
  targets    = local.file_match.traefik.targets
  forward_to = [loki.process.traefik.receiver]
}

loki.process "traefik" {
  stage.static_labels {
    values = {
      job = "traefik",
    }
  }

  forward_to = [loki.write.default.receiver]
}
```

**Note**: Ensure your Alloy `docker-compose.yml` has the Traefik log directory mounted:

```yaml title="docker-compose.yml"
volumes:
  - ./config.alloy:/etc/alloy/config.alloy:ro
  - /var/log:/var/log:ro  # This allows access to /var/log/traefik
  - alloy-data:/var/lib/alloy/data
```

After updating the configuration, restart the Alloy service for the changes to take effect:

```bash
docker restart alloy
```

## Grafana Dashboard

Now you can visualize the collected metrics and logs in Grafana.

### Creating Your Dashboard

You can either create a custom dashboard or use a pre-built one:

- **Traefik Metrics Documentation**: See the [official metrics overview][here] for available metrics
- **Pre-built Dashboard**: Use the ready-made [Traefik Dashboard] from GitHub

Your monitoring setup is now complete. The dashboard provides insights into your Traefik instance's performance and helps identify issues quickly.

[grafana-prometheus]: ../host-container-monitoring
[alloy-loki]: ../system-logs-loki
[here]: https://doc.traefik.io/traefik/observability/metrics/overview/#global-metrics
[traefik dashboard]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/traefik
