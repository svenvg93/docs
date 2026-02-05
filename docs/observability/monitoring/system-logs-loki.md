---
title: System Logs
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

Loki is required to store and query the logs collected by Alloy.

!!! info "Loki Installation"
    If you haven't installed Loki yet, follow the [Install Loki](../tools/install-loki.md) guide first.

## Setup Grafana Alloy

To ship logs to Loki, you need Grafana Alloy installed and configured with the Loki endpoint.

!!! info "Alloy Installation"
    If you haven't installed Alloy yet, follow the [Install Alloy](../tools/install-alloy.md) guide first. Make sure your `endpoint.alloy` includes the Loki endpoint configuration.

### Add System Logs Configuration

Create a new config file for system log collection:

```bash
nano alloy/config/syslog.alloy
```

Paste the following configuration:

```hcl title="syslog.alloy"
// System auth.log collection
local.file_match "authlog" {
  path_targets = [{
    __path__ = "/var/log/auth.log", // (1)!
  }]
}

loki.source.file "authlog" {
  targets    = local.file_match.authlog.targets
  forward_to = [loki.process.authlog.receiver]
}

loki.process "authlog" {
  stage.static_labels {
    values = {
      job = "authlog", // (2)!
    }
  }

  forward_to = [loki.write.default.receiver] // (3)!
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
```

1. :material-file-search: **Log Path** - Adjust to match your log file locations. Supports wildcards like `/var/log/*.log`
2. :material-tag: **Job Label** - Custom label to identify this log source in Grafana
3. :material-arrow-right: **Forward To** - References the Loki endpoint defined in `endpoint.alloy`

!!! note "Log folder access"
    Alloy needs access to the host's log directory. Add the following volume mount to your Alloy `docker-compose.yml`:

    ```yaml
    volumes:
      - /var/log:/var/log:ro
    ```

**Configuration explained:**

- **local.file_match**: Defines which log files to monitor (supports wildcards like `/var/log/*.log`)
- **loki.source.file**: Creates a file reader that tails log files and tracks reading positions
- **loki.process**: Processes logs and adds labels for organization in Grafana
- **stage.static_labels**: Adds custom labels to identify the log source (like `job: "syslog"`)

### Start Services

Restart Alloy to pick up the new configuration:

```bash
docker restart alloy
```

## Grafana

!!! info "Loki Datasource"
    Make sure you have added Loki as a datasource in Grafana. See the [Install Loki](../tools/install-loki.md#grafana-datasource) guide for instructions.

### Exploring Logs in Grafana

Now that you have added Loki as a datasource, you can explore your logs:

1. In the left sidebar, click on Explore.
2. In the top-left dropdown menu, choose Loki as your datasource.
3. In the query section, select the label filename and set the value to /logs/syslog

## Summary

With Loki configured as a datasource in Grafana, Alloy will continuously collect and send log files to Loki, allowing you to visualize and analyze logs easily. This setup provides a comprehensive monitoring solution, enabling you to monitor both metrics and logs from your applications.

Grafana Alloy's modern architecture provides better performance and more flexible log processing compared to Promtail, while integrating seamlessly with the Grafana ecosystem.

[grafana-setup]: ./host-container-monitoring
