---
title: Monitoring Setup
description: Monitoring VyOS router with Prometheus and Loki
tags:
- vyos
- firewall
---

VyOS has built-in support for Prometheus exporters and Syslog log forwarding, making it easy to integrate your router into an existing monitoring stack. This guide covers exposing system metrics via Node Exporter and shipping logs to Loki.

??? info "Prerequisites"
    This guide assumes you already have Grafana, Prometheus, Alloy, and Loki set up. See the following guides:

    - [Install Alloy] - Telemetry collector setup
    - [Install Loki] - Log aggregation setup
    - [Install Prometheus] - Metrics storage
    - [Install Grafana] - Visualization


## Metrics

### VyOS

Enable the built-in Node Exporter on VyOS to expose system metrics such as CPU, memory, disk, and network statistics.

```bash
set service monitoring prometheus node-exporter listen-address 192.168.1.1 # (1)!
set service monitoring prometheus node-exporter port 9100 # (2)!
```

1. The IP address or hostname of the router’s Prometheus metrics endpoint
2. The TCP port used by the router to expose Prometheus metrics

### Prometheus
To ensure Prometheus collects metrics from your VyOS router, you need to add a scrape configuration to your existing Prometheus configuration.

Add the following configuration to your `prometheus.yml`:
```yaml title="prometheus.yml" hl_lines="5"
scrape_configs:
  - job_name: 'VyOS'
    scrape_interval: 5s
    static_configs:
      - targets: ['<router-ip>:9100'] # (1)!
```

1. Replace `<router-ip>:9100` with your router IP and metrics port.

To apply the configuration changes, restart the Prometheus containers:

```bash
docker restart prometheus
```

### Internet Connection Montioring

Todo

Internet connection monitoring
https://docs.vyos.io/en/latest/configuration/service/monitoring.html#blackbox-exporter

## Logs

VyOS can forward its syslog messages to an external log processor such as Grafana Alloy. Alloy can then parse, label, and forward these logs to Loki, allowing you to centralize router logs alongside the rest of your infrastructure in Grafana.

### VyOS

Enable syslog forwarding on VyOS to send system and service logs to an external collector.

```bash hl_lines="3 4 5"
set system syslog local facility all level 'info' # (1)!
set system syslog local facility local7 level 'debug' # (2)!
set system syslog remote 192.168.1.101 facility all level 'all' # (3)!
set system syslog remote 192.168.1.101 port '514' # (4)!
set system syslog remote 192.168.1.101 protocol 'udp' # (5)!
commit; save
```

1. Logs all facilities at info level locally
2. Logs local7 facility at debug level for detailed diagnostics
3. Forwards all log facilities to the remote syslog collector
4. Port of the remote syslog collector
5. Uses UDP as the transport protocol

### Grafana Alloy

Create a new Alloy config file for Traefik log collection:

```bash
nano alloy/config/vyos-syslog.alloy
```

```hcl title="vyos-syslog.alloy"
/* VyOS Syslog (RFC3164) */
loki.relabel "vyos_syslog" {
  forward_to = []

  // Map hostname from syslog header
  rule {
    source_labels = ["__syslog_message_hostname"]
    target_label  = "host"
  }

  // Map application name
  rule {
    source_labels = ["__syslog_message_app_name"]
    target_label  = "service_name"
  }

  // Map syslog severity level
  rule {
    source_labels = ["__syslog_message_severity"]
    target_label  = "detected_level"
  }
}

loki.source.syslog "vyos" {
  listener {
    address       = "0.0.0.0:514"
    protocol      = "udp"
    syslog_format = "rfc3164"
    use_incoming_timestamp = false
    labels        = {
      job      = "vyos",
    }
  }

  relabel_rules = loki.relabel.vyos_syslog.rules
  forward_to    = [loki.write.default.receiver]
}
```

Restart Alloy to pick up the new configuration:

```bash
docker restart alloy
```

### Creating Your Dashboard

Coming Soon!

[Install Alloy]: ../../../observability/tools/install-alloy.md
[Install Loki]: ../../../observability/tools/install-loki.md
[Install Prometheus]: ../../../observability/tools/install-prometheus.md
[Install Grafana]: ../../../observability/tools/install-grafana.md