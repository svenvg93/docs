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


## System Metrics

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
  - job_name: 'vyos'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.1.1:9100'] # (1)!
        labels:
         router: vyos # (2)!
```

1. Replace `192.168.1.1:9100` with your router IP and metrics port.
2. Label to indetify the system across the diffrent scrape jobs

To apply the configuration changes, restart the Prometheus containers:

```bash
docker restart prometheus
```

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

## Connection Metrics

VyOS includes a built-in Blackbox Exporter that can be used to collect metrics about your internet connection, such as reachability and latency.

### VyOS

Enable blackbox exporter, so Prometheus can scrape the test resutls. In this example we only ass ICMP testing 

```bash
set service monitoring prometheus blackbox-exporter listen-address '192.168.1.1'
set service monitoring prometheus blackbox-exporter modules dns
set service monitoring prometheus blackbox-exporter modules icmp name icmp preferred-ip-protocol ipv4
```

### Prometheu

Enable the Blackbox Exporter so Prometheus can scrape the probe results.In this example, we configure ICMP (ping) checks only to measure basic connectivity and latency.
Add the following configuration to your `prometheus.yml`:

```yaml title="prometheus.yml" hl_lines="6 7 8 9 10 11 12 13 14 15 23 27 "
  - job_name: 'blackbox_icmp'
    scrape_interval: 15s
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets: ['connectivitycheck.gstatic.com'] # (1)!
        labels:
          endpoint: "Google"
      - targets: ['dualstack.nonssl.global.fastly.net']
        labels:
          endpoint: "Fastly"
      - targets: ['workers.dev']
        labels:
          endpoint: "Cloudflare"
    relabel_configs:
      # Set the target for blackbox
      - source_labels: [__address__]
        target_label: __param_target

      # Set the instance to match your router hostname
      - target_label: router # (2_ !
        replacement: vyos

      # Point scrape to blackbox exporter itself
      - target_label: __address__
        replacement: 192.168.1.1:9115 # (3)!
```

1. Target for Blackbox Exporter to ping.
2. Label to indetify the system across the diffrent scrape jobs.
3. Replace `192.168.1.1:9115` with your router IP and metrics port.

## Grafana Dashboard

Now you can visualize the collected metrics and logs in Grafana.

### Creating Your Dashboard

You can either create a custom dashboard or use a pre-built one:

- **Pre-built Dashboard**: Use the ready-made [VyOS Dashboard] from GitHub

Your monitoring setup is now complete. The dashboard provides insights into your Traefik instance's performance and helps identify issues quickly
[Install Alloy]: ../../../observability/tools/install-alloy.md
[Install Loki]: ../../../observability/tools/install-loki.md
[Install Prometheus]: ../../../observability/tools/install-prometheus.md
[Install Grafana]: ../../../observability/tools/install-grafana.md
[VyOS Dashboard]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/vyos
