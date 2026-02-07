---
title: Host & Container Metrics
description: Learn how to set up performance monitoring for your systems and containers using Grafana Alloy, Prometheus, and Grafana.
tags:
- docker
- prometheus
- grafana
- alloy
---

Monitoring your systems and containers is essential for maintaining a reliable homelab or home server. This guide uses Grafana Alloy to collect both host and container metrics, with Prometheus for storage and Grafana for visualization.

## Key Components

1. Grafana Alloy - Collects host metrics (CPU, memory, disk, network) and container metrics from Docker.
2. Prometheus - Stores time-series metrics received from Alloy.
3. Grafana - Provides visualization dashboards to analyze and monitor your system and container metrics.

??? info "Prerequisites"
    Ensure you have the following installed before proceeding:

    - [Install Alloy] - Telemetry collector
    - [Install Prometheus] - Metrics storage
    - [Install Grafana] - Visualization

## Grafana Alloy Configuration

Add the following configuration files to your Alloy `config/` directory to enable host and container metrics collection.

### Host Metrics

Create an Alloy config file for host system metrics:

```bash
nano alloy/config/unix.alloy
```

```hcl title="unix.alloy"
// Host system metrics (like Node Exporter)
prometheus.exporter.unix "host" {
  set_collectors = ["cpu", "diskstats", "filesystem", "loadavg", "meminfo", "netdev", "uname"] // (1)!
  filesystem {
    mount_points_exclude = "^/(sys|proc|dev|host|etc)($$|/)" // (2)!
  }
  netdev {
    device_exclude = "^(veth.*|br.*|docker.*|virbr.*|lo)$$" // (3)!
  }
}

prometheus.scrape "host" {
  targets    = prometheus.exporter.unix.host.targets
  forward_to = [prometheus.remote_write.default.receiver]
  job_name   = "host" // (4)!
}
```

1. :material-chip: **Collectors** - Select which metrics to collect. Common options: `cpu`, `diskstats`, `filesystem`, `loadavg`, `meminfo`, `netdev`, `uname`, `time`
2. :material-folder-off: **Filesystem Exclude** - Ignores virtual filesystems to avoid cluttering metrics
3. :material-network-off: **Network Exclude** - Excludes virtual network interfaces (Docker bridges, veth pairs, etc.)
4. :material-tag: **Job Name** - Label to identify these metrics in Grafana

### Container Metrics

Create an Alloy config file for Docker container metrics:

```bash
nano alloy/config/docker.alloy
```

```hcl title="docker.alloy"
// Docker container metrics (like cAdvisor)
prometheus.exporter.cadvisor "containers" {
  docker_host = "unix:///var/run/docker.sock" // (1)!
  store_container_labels = false // (2)!
}

prometheus.scrape "containers" {
  targets    = prometheus.exporter.cadvisor.containers.targets
  forward_to = [prometheus.remote_write.default.receiver]
  job_name   = "cadvisor" // (3)!
}
```

1. :material-docker: **Docker Socket** - Path to Docker socket for container access
2. :material-label-off: **Container Labels** - Set to `true` if you want Docker labels as Prometheus labels
3. :material-tag: **Job Name** - Label to identify container metrics in Grafana

!!! note "Docker socket access"
    For Docker installations, ensure Alloy has access to the Docker socket. In the Alloy `docker-compose.yml`, add:

    ```yaml
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ```

??? warning "Run as root for Docker access"
    When running Alloy via systemd to collect Docker container logs or metrics, the service needs to run as root to access the Docker socket. Edit the systemd service:

    ```bash
    sudo systemctl edit alloy
    ```

    Add the following override:

    ```ini
    [Service]
    User=root
    Group=root
    ```

    Then restart: `sudo systemctl restart alloy`

    See [GitHub issue #4068] for more details.

## Apply Configuration

Restart Alloy to apply the new configurations:

```bash
docker restart alloy
```

## Grafana Dashboards

Import dashboards to visualize your metrics:

- [Node Exporter Dashboard][nodeexporter] - Host system metrics
- [cAdvisor Dashboard][cadvisor] - Container metrics

## Verification

Verify metrics are being collected:

**Check Alloy components:**

```bash
docker logs alloy | grep -i "unix\|cadvisor"
```

## Conclusion

You now have host and container monitoring using Grafana Alloy. This setup replaces the need for separate Node Exporter and cAdvisor containers, consolidating metrics collection into a single Alloy instance.

[Install Alloy]: ../tools/install-alloy.md
[Install Prometheus]: ../tools/install-prometheus.md
[Install Grafana]: ../tools/install-grafana.md
[GitHub issue #4068]: https://github.com/grafana/alloy/issues/4068
[nodeexporter]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/node_expoter
[cadvisor]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/cadvisor
