---
title: Monitoring Setup
description: Monitor your MikroTik router with Grafana Alloy using SNMP metrics and syslog forwarding.
tags:
- mikrotik
- snmp
- alloy
- grafana
---

MikroTik routers support SNMP and syslog, making it straightforward to collect metrics and logs using Grafana Alloy. This guide covers enabling SNMP on the router for metrics collection via Alloy's built-in SNMP exporter, and forwarding syslog messages to Loki.

??? info "Prerequisites"
    This guide assumes you already have Grafana, Alloy, and Loki set up. See the following guides:

    - [Install Alloy] - Telemetry collector setup
    - [Install Loki] - Log aggregation setup
    - [Install Grafana] - Visualization

## System Metrics

MikroTik does not have a built-in Prometheus exporter, but it does support SNMP. Grafana Alloy includes a built-in [SNMP exporter] that can poll your router and expose metrics as Prometheus-compatible data.

### MikroTik

Enable SNMP on the router and restrict it to the LAN interface for security:

```bash hl_lines="2 3"
/snmp
set enabled=yes contact="" location=""
set trap-community=public trap-version=2
/snmp community
set public addresses=192.168.1.0/24 read-access=yes write-access=no # (1)!
```

1. :material-lock: **Access Control** - Restricts SNMP access to your LAN subnet only. Adjust the address range to match your network.

### Grafana Alloy

Create a new Alloy config file for MikroTik SNMP collection:

```bash
nano alloy/config/mikrotik-snmp.alloy
```

```hcl title="mikrotik-snmp.alloy" hl_lines="5 6 11"
// MikroTik SNMP metrics
prometheus.exporter.snmp "mikrotik" {
  config_file = "/etc/alloy/config/snmp.yml" // (1)!

  target "router" {
    address     = "192.168.1.1" // (2)!
    module      = "mikrotik" // (3)!
    walk_params = "mikrotik"
  }

  walk_param "mikrotik" {
    retries        = 3
    timeout        = "10s" // (4)!
    max_repetitions = 25
  }
}

prometheus.scrape "mikrotik" {
  targets    = prometheus.exporter.snmp.mikrotik.targets
  forward_to = [prometheus.remote_write.default.receiver]
  job_name   = "mikrotik" // (5)!
  scrape_interval = "60s" // (6)!
}
```

1. :material-file-cog: **SNMP Config** - Path to the SNMP module definition file (see below)
2. :material-router-network: **Router IP** - Replace with your MikroTik router's IP address
3. :material-puzzle: **Module** - References the `mikrotik` module defined in `snmp.yml`
4. :material-timer-outline: **Timeout** - SNMP walk timeout, increase if your router has many interfaces
5. :material-tag: **Job Name** - Label to identify MikroTik metrics in Grafana
6. :material-clock-outline: **Scrape Interval** - How often to poll the router. SNMP walks can be resource-intensive, so 60s is a good starting point

### SNMP Module Configuration

The SNMP exporter requires a module definition file that specifies which OIDs to collect. Create the `snmp.yml` file:

```bash
nano alloy/config/snmp.yml
```

```yaml title="snmp.yml"
auths:
  mikrotik_v2:
    community: public # (1)!
    version: 2

modules:
  mikrotik:
    walk:
      - sysUpTime # (2)!
      - interfaces # (3)!
      - ifXTable # (4)!
      - 1.3.6.1.2.1.25.3.3.1.2 # (5)!
      - 1.3.6.1.2.1.25.2 # (6)!
      - 1.3.6.1.4.1.14988.1.1.3 # (7)!
    metrics:
      - name: sysUpTime
        oid: 1.3.6.1.2.1.1.3
        type: gauge
        help: System uptime in hundredths of a second.

      - name: ifNumber
        oid: 1.3.6.1.2.1.2.1
        type: gauge
        help: Number of network interfaces.

      - name: ifHCInOctets
        oid: 1.3.6.1.2.1.31.1.1.1.6
        type: counter
        help: Total bytes received on the interface.
        indexes:
          - labelname: ifIndex
            type: Integer
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString
          - labels: [ifIndex]
            labelname: ifAlias
            oid: 1.3.6.1.2.1.31.1.1.1.18
            type: DisplayString

      - name: ifHCOutOctets
        oid: 1.3.6.1.2.1.31.1.1.1.10
        type: counter
        help: Total bytes transmitted on the interface.
        indexes:
          - labelname: ifIndex
            type: Integer
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString
          - labels: [ifIndex]
            labelname: ifAlias
            oid: 1.3.6.1.2.1.31.1.1.1.18
            type: DisplayString

      - name: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
        help: "Operational status of the interface: up(1), down(2)."
        indexes:
          - labelname: ifIndex
            type: Integer
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: hrProcessorLoad
        oid: 1.3.6.1.2.1.25.3.3.1.2
        type: gauge
        help: CPU load percentage per core.
        indexes:
          - labelname: hrDeviceIndex
            type: Integer

      - name: hrStorageUsed
        oid: 1.3.6.1.2.1.25.2.3.1.6
        type: gauge
        help: Amount of storage used (in allocation units).
        indexes:
          - labelname: hrStorageIndex
            type: Integer
        lookups:
          - labels: [hrStorageIndex]
            labelname: hrStorageDescr
            oid: 1.3.6.1.2.1.25.2.3.1.3
            type: DisplayString

      - name: hrStorageSize
        oid: 1.3.6.1.2.1.25.2.3.1.5
        type: gauge
        help: Total storage size (in allocation units).
        indexes:
          - labelname: hrStorageIndex
            type: Integer
        lookups:
          - labels: [hrStorageIndex]
            labelname: hrStorageDescr
            oid: 1.3.6.1.2.1.25.2.3.1.3
            type: DisplayString

      - name: hrStorageAllocationUnits
        oid: 1.3.6.1.2.1.25.2.3.1.4
        type: gauge
        help: Size of an allocation unit in bytes.
        indexes:
          - labelname: hrStorageIndex
            type: Integer
        lookups:
          - labels: [hrStorageIndex]
            labelname: hrStorageDescr
            oid: 1.3.6.1.2.1.25.2.3.1.3
            type: DisplayString

      - name: mtxrHlTemperature
        oid: 1.3.6.1.4.1.14988.1.1.3.10
        type: gauge
        help: Router board temperature in degrees Celsius (x0.1).

      - name: mtxrHlVoltage
        oid: 1.3.6.1.4.1.14988.1.1.3.8
        type: gauge
        help: Router board voltage in volts (x0.1).

    auth:
      community: public
      version: 2
```

1. :material-key: **Community String** - Must match the community configured on the router
2. :material-clock-outline: **System Uptime** - Tracks how long the router has been running
3. :material-lan: **Interfaces** - Standard interface statistics (status, speed, errors)
4. :material-speedometer: **Extended Interfaces** - 64-bit traffic counters for high-throughput interfaces
5. :material-chip: **CPU Load** - Per-core CPU utilization from HOST-RESOURCES-MIB
6. :material-memory: **Storage/Memory** - Memory and disk usage from HOST-RESOURCES-MIB
7. :material-thermometer: **MikroTik Health** - MikroTik-specific OIDs for board temperature and voltage

!!! tip "Docker volume mount"
    Ensure the `snmp.yml` file is accessible inside the Alloy container. If you followed the [Install Alloy] guide, files in the `config/` directory are already mounted at `/etc/alloy/config/`.

## Logs

MikroTik supports remote syslog logging, allowing you to forward system events, firewall logs, and service messages to Grafana Alloy for centralized log management in Loki.

### MikroTik

Enable remote syslog forwarding on the router:

```bash hl_lines="2 3"
/system logging action
set remote address=192.168.1.101 port=1514 remote-port=1514 bsd-syslog=yes syslog-facility=local0 # (1)!
/system logging
add action=remote topics=info # (2)!
add action=remote topics=warning
add action=remote topics=error
add action=remote topics=critical
```

1. :material-server-network: **Remote Address** - Replace `192.168.1.101` with the IP of your Alloy instance and `1514` with the port Alloy is listening on
2. :material-filter: **Log Topics** - Add or remove topics to control what gets forwarded. Common topics: `info`, `warning`, `error`, `critical`, `firewall`, `dhcp`, `dns`, `wireless`

??? tip "Log firewall activity"
    To include firewall logs, add the `firewall` topic:

    ```bash
    /system logging
    add action=remote topics=firewall
    ```

    You also need firewall rules that use the `log=yes` option:

    ```bash
    /ip firewall filter
    set [find comment="drop all else" chain=forward] log=yes log-prefix="FW-DROP-FWD"
    set [find comment="Drop all else" chain=input] log=yes log-prefix="FW-DROP-IN"
    ```

### Grafana Alloy

Create a new Alloy config file for MikroTik syslog collection:

```bash
nano alloy/config/mikrotik-syslog.alloy
```

```hcl title="mikrotik-syslog.alloy"
/* MikroTik Syslog (BSD/RFC3164) */
loki.relabel "mikrotik_syslog" {
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

loki.source.syslog "mikrotik" {
  listener {
    address            = "0.0.0.0:1514" // (4)!
    protocol           = "udp"
    syslog_format      = "rfc3164"
    use_incoming_timestamp = false
    labels             = {
      job = "mikrotik",
    }
  }

  relabel_rules = loki.relabel.mikrotik_syslog.rules
  forward_to    = [loki.write.default.receiver] // (5)!
}
```

1. :material-server: **Host Label** - Maps the syslog hostname to the `host` label
2. :material-application: **Service Name** - Maps the syslog application to the `service_name` label
3. :material-alert: **Log Level** - Maps the syslog severity to the `detected_level` label
4. :material-network: **Listen Port** - The UDP port Alloy listens on. Must match the port configured on the MikroTik router
5. :material-arrow-right: **Forward To** - References the Loki endpoint defined in `endpoint.alloy`

## Apply Configuration

Restart Alloy to load the new configuration files:

```bash
docker restart alloy
```

## Verification

After restarting Alloy, verify that both metrics and logs are flowing.

**Check Alloy components:**

Open the Alloy Web UI and verify the `prometheus.exporter.snmp` and `loki.source.syslog` components are healthy.

**Verify SNMP metrics in Grafana:**

Query for MikroTik metrics in Grafana's Explore view:

```promql
ifHCInOctets{job="mikrotik"}
```

**Verify logs in Grafana:**

Query for MikroTik logs in Grafana's Explore view using LogQL:

```logql
{job="mikrotik"}
```

[Install Alloy]: ../../../observability/tools/install-alloy.md
[Install Loki]: ../../../observability/tools/install-loki.md
[Install Grafana]: ../../../observability/tools/install-grafana.md
[SNMP exporter]: https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.snmp/
