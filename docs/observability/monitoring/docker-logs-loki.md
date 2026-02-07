---
title: Docker Container Logs
description: Collect and visualize Docker container logs using Grafana Alloy and Loki.
tags:
- docker
- grafana
- loki
- alloy
---

Centralize your Docker container logs with Loki and Grafana Alloy. This guide configures Alloy to collect logs from all running containers and forward them to Loki for querying and visualization in Grafana.

## Key Components

1. Loki - Log aggregation system that stores and indexes logs efficiently for querying and analysis.
2. Grafana Alloy - Telemetry collector that discovers and ships Docker container logs to Loki.
3. Grafana - Provides visualization and search capabilities for logs stored in Loki.

??? info "Prerequisites"
    Ensure you have the following installed before proceeding:

    - [Install Alloy] - Telemetry collector
    - [Install Loki] - Log aggregation
    - [Install Grafana] - Visualization

## Grafana Alloy Configuration

Add the following configuration file to your Alloy `config/` directory to enable Docker log collection.

### Docker Logs Configuration

Create an Alloy config file for Docker container logs:

```bash
nano alloy/config/docker-logs.alloy
```

Paste the following configuration:

```hcl title="docker-logs.alloy"
// Discover Docker containers
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock" // (1)!
}

// Relabel discovered containers
discovery.relabel "containers" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    target_label  = "container" // (2)!
    regex         = "/(.*)"
    replacement   = "$1"
  }

  rule {
    source_labels = ["__meta_docker_container_log_stream"]
    target_label  = "stream" // (3)!
  }
}

// Collect logs from discovered containers
loki.source.docker "containers" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.containers.output
  forward_to = [loki.process.containers.receiver]
}

// Process and label logs
loki.process "containers" {
  stage.static_labels {
    values = {
      job = "docker", // (4)!
    }
  }

  forward_to = [loki.write.default.receiver] // (5)!
}
```

1. :material-docker: **Docker Socket** - Path to Docker socket for container discovery
2. :material-tag: **Container Label** - Extracts container name (removes leading slash)
3. :material-format-list-bulleted: **Stream Label** - Captures stdout/stderr stream type
4. :material-tag: **Job Label** - Groups all container logs under the "docker" job
5. :material-arrow-right: **Forward To** - References the Loki endpoint defined in `endpoint.alloy`

!!! note "Docker socket access"
    Alloy needs access to the Docker socket. Add the following volume mount to your Alloy `docker-compose.yml`:

    ```yaml
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ```

??? warning "Run as root for Docker access"
    When running Alloy via systemd, the service needs to run as root to access the Docker socket. Edit the systemd service:

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

**Configuration explained:**

- **discovery.docker**: Automatically discovers all running Docker containers
- **discovery.relabel**: Extracts metadata like container name and stream type as labels
- **loki.source.docker**: Connects to Docker and collects logs from discovered containers
- **loki.process**: Adds a static `job` label and forwards to Loki

??? tip "Filter specific containers"
    To collect logs from specific containers only, add a filter rule:

    ```hcl
    discovery.relabel "containers" {
      targets = discovery.docker.containers.targets

      // Only include containers with specific names
      rule {
        source_labels = ["__meta_docker_container_name"]
        regex         = "/(nginx|traefik|api)"
        action        = "keep"
      }

      // ... rest of rules
    }
    ```

## Apply Configuration

Restart Alloy to apply the new configuration:

```bash
docker restart alloy
```

## Grafana

!!! info "Loki Datasource"
    Make sure you have added Loki as a datasource in Grafana. See the [Install Loki] guide for instructions.

### Exploring Logs in Grafana

Query your Docker container logs in Grafana:

1. In the left sidebar, click on **Explore**
2. Select **Loki** as your datasource
3. Use LogQL to query logs:

```logql
{job="docker"}
```

**Filter by container:**

```logql
{job="docker", container="nginx"}
```

**Search for errors:**

```logql
{job="docker"} |= "error"
```

## Summary

You now have centralized Docker container logging with Grafana Alloy and Loki. All container logs are automatically discovered and collected, making it easy to search and analyze logs across your entire Docker environment.

[Install Alloy]: ../tools/install-alloy.md
[Install Loki]: ../tools/install-loki.md
[Install Grafana]: ../tools/install-grafana.md
[GitHub issue #4068]: https://github.com/grafana/alloy/issues/4068
