---
title: Unifi Syslog
description: Send Unifi Syslog to Loki with Alloy
tags:
- unifi
- loki
- logs
- alloy
---

Unifi network devices generate valuable syslog messages that can help you troubleshoot network issues and monitor your devices. This guide covers forwarding these logs to Loki using Grafana Alloy, allowing you to centralize your network logs alongside the rest of your infrastructure in Grafana.

This guide focuses on device logs only. Security and firewall logs are out of scope.

??? info "Prerequisites"
    This guide assumes you already have Grafana, Alloy, and Loki set up. See the following guides:

    - [Install Alloy] - Telemetry collector setup
    - [Install Loki] - Log aggregation setup
    - [Install Grafana] - Visualization

## Logs

### Unifi Controller

Configure your Unifi Controller to forward syslog messages to Alloy.

1. Open your Unifi Controller
2. Navigate to **Settings** > **Cyber Secure** > **Traffic Logging**
3. Set **Activity Logging (Syslog)** to **SIEM Server**
4. Set the **IP Address** to your Alloy server IP (e.g., `192.168.1.100`)
5. Set the **Port** to `514`
6. Click **Apply Changes**

### Grafana Alloy

Create a new Alloy config file for Unifi log collection:

```bash
nano alloy/config/unifi-syslog.alloy
```

```hcl title="unifi-syslog.alloy"
/* UniFi Syslog (RFC3164) - Relabel rules to capture syslog metadata */
loki.relabel "unifi_syslog" {
  forward_to = []

  // Map syslog severity number to named level
  rule {
    source_labels = ["__syslog_message_severity"]
    target_label  = "detected_level"
  }

  // Map hostname from syslog header
  rule {
    source_labels = ["__syslog_message_hostname"]
    target_label  = "host"
  }
}

loki.source.syslog "unifi" {
  listener {
    address       = "0.0.0.0:514"
    protocol      = "udp"
    syslog_format = "rfc3164"
    labels        = {
      job      = "unifi",
    }
  }

  relabel_rules = loki.relabel.unifi_syslog.rules
  forward_to    = [loki.process.unifi.receiver]
}

loki.process "unifi" {
  // Extract app and message from UniFi syslog content
  // Format 1 (AP/Switch): "mac,device-firmware: process[pid]: message"
  // Format 2 (Gateway): "hostname process[pid]: message"
  stage.regex {
    expression = "^(?:[\\w,\\-\\.\\+]+:\\s+|[\\w\\-]+\\s+)?(?P<service_name>[\\w\\-]+)(?:\\[\\d+\\])?:\\s*(?P<message>.*)$"
  }

  stage.labels {
    values = {
      detected_level = "",
      service_name   = "",
    }
  }

  // Output just the message content
  stage.output {
    source = "message"
  }

  forward_to = [loki.write.default.receiver]
}
```

1. Maps the syslog severity to the `detected_level` label, with normalization rules to standardize values (e.g., `emergency` → `critical`, `notice` → `info`).
2. Maps the syslog hostname to the `host` label (e.g., `UCG-Fiber`, `U7-Pro-Wall`).
3. Unifi devices use RFC3164 syslog format over UDP.
4. Outputs only the message content, removing metadata already captured as labels.

!!! note "Syslog port"
    Ensure your Alloy `docker-compose.yml` exposes UDP port 514:

    ```yaml
    ports:
      - "514:514/udp"
    ```

Restart Alloy to pick up the new configuration:

```bash
docker restart alloy
```

## Verification

After configuring your Unifi devices, logs should start flowing to Loki within seconds.

1. Open Grafana and navigate to **Explore**
2. Select **Loki** as the datasource
3. Run a query to see Unifi logs:

```
{job="unifi"}
```

You should see syslog messages from your Unifi devices, including authentication events, DHCP assignments, and wireless connections.

??? tip "Useful Queries"
    **Logs from a specific device:**
    ```
    {job="unifi", host="UCG-Fiber"}
    ```

    **Logs from a specific application:**
    ```
    {job="unifi", app="hostapd"}
    ```

## Grafana Dashboard

Now you can visualize the collected logs in Grafana.

### Creating Your Dashboard

You can either create a custom dashboard or use a pre-built one:

- **Pre-built Dashboard**: Use the ready-made [Unifi Syslog Dashboard] from GitHub

Your logging setup is now complete. The dashboard provides insights into your Unifi network devices and helps identify issues quickly.

[Install Alloy]: ../../../observability/tools/install-alloy.md
[Install Loki]: ../../../observability/tools/install-loki.md
[Install Grafana]: ../../../observability/tools/install-grafana.md
[Unifi Syslog Dashboard]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/unifi
