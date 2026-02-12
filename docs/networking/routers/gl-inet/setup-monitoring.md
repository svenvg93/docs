---
title: Monitoring Setup
description: Monitor your GL.iNet router with Grafana Alloy using Prometheus metrics and syslog forwarding.
tags:
- gl-inet
- prometheus
- alloy
- grafana
---

GL.iNet routers run OpenWRT, which has native [Prometheus exporter] packages available. This guide covers installing the Prometheus node exporter on the router for metrics collection, and forwarding syslog messages to Loki via Grafana Alloy.

??? info "Prerequisites"
    This guide assumes you already have Grafana, Alloy, Prometheus, and Loki set up. See the following guides:

    - [Install Alloy] - Telemetry collector setup
    - [Install Loki] - Log aggregation setup
    - [Install Prometheus] - Metrics storage
    - [Install Grafana] - Visualization

## System Metrics

OpenWRT provides a lightweight Lua-based Prometheus node exporter with modular collectors for CPU, memory, network, Wi-Fi, and more.

### GL.iNet

SSH into your GL.iNet router and install the exporter packages:

```bash
opkg update
opkg install prometheus-node-exporter-lua \
  prometheus-node-exporter-lua-openwrt \
  prometheus-node-exporter-lua-wifi \
  prometheus-node-exporter-lua-wifi_stations \
  prometheus-node-exporter-lua-netstat \
  prometheus-node-exporter-lua-nat_traffic \
  prometheus-node-exporter-lua-thermal \
  prometheus-node-exporter-lua-hwmon \
  prometheus-node-exporter-lua-uci_dhcp_host
```

??? note "Available collector packages"
    | Package | Description |
    | ------- | ----------- |
    | `prometheus-node-exporter-lua` | Core exporter with CPU, memory, disk, and network metrics |
    | `prometheus-node-exporter-lua-openwrt` | OpenWRT-specific metrics |
    | `prometheus-node-exporter-lua-wifi` | Wi-Fi interface statistics |
    | `prometheus-node-exporter-lua-wifi_stations` | Connected Wi-Fi client metrics |
    | `prometheus-node-exporter-lua-netstat` | TCP/UDP connection statistics |
    | `prometheus-node-exporter-lua-nat_traffic` | NAT traffic counters |
    | `prometheus-node-exporter-lua-thermal` | Thermal zone temperatures |
    | `prometheus-node-exporter-lua-hwmon` | Hardware sensor readings |
    | `prometheus-node-exporter-lua-uci_dhcp_host` | DHCP host information |

    Install only the packages relevant to your setup. The core package is required, the rest are optional.

By default, the exporter only listens on localhost. Update it to listen on the LAN interface so Prometheus can scrape it:

```bash hl_lines="1 2"
uci set prometheus-node-exporter-lua.main.listen_interface='lan' # (1)!
uci set prometheus-node-exporter-lua.main.listen_port='9100' # (2)!
uci commit prometheus-node-exporter-lua
/etc/init.d/prometheus-node-exporter-lua enable
/etc/init.d/prometheus-node-exporter-lua restart
```

1. :material-lan: **Listen Interface** - Exposes metrics on the LAN interface. Change to a specific interface name if needed
2. :material-network: **Port** - The TCP port the exporter listens on

??? tip "Verify the exporter is working"
    From another machine on the same network, test the metrics endpoint:

    ```bash
    curl http://192.168.8.1:9100/metrics
    ```

    You should see Prometheus-formatted metrics output.

### Prometheus

Add a scrape configuration for the GL.iNet router to your `prometheus.yml`:

```yaml title="prometheus.yml" hl_lines="4 6"
  - job_name: 'glinet'
    scrape_interval: 15s
    static_configs:
      - targets: ['192.168.8.1:9100'] # (1)!
        labels:
          router: glinet # (2)!
```

1. :material-router-network: **Target** - Replace `192.168.8.1:9100` with your router IP and metrics port
2. :material-tag: **Label** - Label to identify the router across scrape jobs

Restart Prometheus to apply the configuration:

```bash
docker restart prometheus
```

## Logs

OpenWRT supports remote syslog logging, allowing you to forward system events and service messages to Grafana Alloy for centralized log management in Loki.

### GL.iNet

You can configure remote syslog either via the LuCI web interface or via SSH.

=== "LuCI Web Interface"

    1. Navigate to **System** > **System** > **Logging**
    2. Set **External system log server** to the IP of your Alloy instance (e.g., `192.168.8.101`)
    3. Set **External system log server port** to `1514`
    4. Set **External system log server protocol** to **UDP**
    5. Set **Log output level** to **Info**
    6. Click **Save & Apply**

=== "SSH (UCI)"

    ```bash hl_lines="1 2 3 4"
    uci set system.@system[0].log_ip='192.168.8.101' # (1)!
    uci set system.@system[0].log_port='514' # (2)!
    uci set system.@system[0].log_proto='udp' # (3)!
    uci set system.@system[0].conloglevel='7' # (4)!
    uci commit system
    /etc/init.d/log restart
    ```

    1. :material-server-network: **Remote Address** - Replace with the IP of your Alloy instance
    2. :material-network: **Port** - The port Alloy is listening on for syslog messages
    3. :material-protocol: **Protocol** - Transport protocol for syslog forwarding
    4. :material-filter: **Log Level** - Log verbosity level (7 = debug, 6 = info, 4 = warning, 3 = error)

### Grafana Alloy

Create a new Alloy config file for GL.iNet syslog collection:

```bash
nano alloy/config/glinet-syslog.alloy
```

```hcl title="glinet-syslog.alloy"
/* GL.iNet Syslog (BSD/RFC3164) */
loki.relabel "glinet_syslog" {
  forward_to = []

  // Map hostname from syslog header
  rule {
    source_labels = ["__syslog_message_hostname"]
    target_label  = "host" // (1)!
  }

  // Map application name
  rule {
    source_labels = ["__syslog_message_app_name"]
    target_label  = "service_name" // (2)!
  }

  // Map syslog severity level
  rule {
    source_labels = ["__syslog_message_severity"]
    target_label  = "detected_level" // (3)!
  }
}

loki.source.syslog "glinet" {
  listener {
    address            = "0.0.0.0:514" // (4)!
    protocol           = "udp"
    syslog_format      = "rfc3164"
    use_incoming_timestamp = false
    labels             = {
      job = "glinet",
    }
  }

  relabel_rules = loki.relabel.glinet_syslog.rules
  forward_to    = [loki.write.default.receiver] // (5)!
}
```

1. :material-server: **Host Label** - Maps the syslog hostname to the `host` label
2. :material-application: **Service Name** - Maps the syslog application to the `service_name` label
3. :material-alert: **Log Level** - Maps the syslog severity to the `detected_level` label
4. :material-network: **Listen Port** - The UDP port Alloy listens on. Must match the port configured on the GL.iNet router
5. :material-arrow-right: **Forward To** - References the Loki endpoint defined in `endpoint.alloy`

## Apply Configuration

Restart Alloy to load the new configuration files:

```bash
docker restart alloy
```

## Verification

After restarting, verify that both metrics and logs are flowing.

**Check Alloy components:**

Open the Alloy Web UI and verify the `loki.source.syslog` component is healthy.

**Verify metrics in Grafana:**

Query for GL.iNet metrics in Grafana's Explore view:

```promql
node_cpu_seconds_total{router="glinet"}
```

**Verify logs in Grafana:**

Query for GL.iNet logs in Grafana's Explore view using LogQL:

```logql
{job="glinet"}
```

[Install Alloy]: ../../../observability/tools/install-alloy.md
[Install Loki]: ../../../observability/tools/install-loki.md
[Install Prometheus]: ../../../observability/tools/install-prometheus.md
[Install Grafana]: ../../../observability/tools/install-grafana.md
[Prometheus exporter]: https://github.com/openwrt/packages/tree/master/utils/prometheus-node-exporter-lua
