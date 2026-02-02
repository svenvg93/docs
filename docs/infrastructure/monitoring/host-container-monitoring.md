---
title: Host & Container Metrics
description: In Part 1 of the System Monitoring series, learn how to set up performance monitoring for your systems and containers using Prometheus and Grafana.
tags:
- docker
- prometheus
- grafana
---

Monitoring your systems and containers is essential for maintaining a reliable homelab or home server. A popular setup involves Prometheus, Node Exporter, and cAdvisor for collecting metrics, combined with Grafana for creating insightful dashboards.

In this guide, we'll set up a complete monitoring solution by:
1.	Configuring Prometheus to scrape metrics from Node Exporter and cAdvisor.
2.	Using Grafana to visualize the data with intuitive dashboards.

Let's dive in and build a robust monitoring stack!

## Key Components

1. Node Exporter - Collects hardware and OS-level metrics from the host system (CPU, memory, disk, network).
2. cAdvisor - Monitors container resource usage and performance metrics for Docker containers.
3. Prometheus - Scrapes and stores time-series metrics from both Node Exporter and cAdvisor.
4. Grafana - Provides visualization dashboards to analyze and monitor your system and container metrics.

!!! info "Prerequisites"
    - Docker and Docker Compose installed on your system
    - A Linux system with systemd (for Node Exporter service management)
    - Root or sudo access for installing Node Exporter

    If you need help setting up Docker, refer to the [official Docker documentation](https://docs.docker.com/get-docker/).

##  Setup Node Exporter

Node Exporter is installed natively on the host system to collect hardware and OS-level metrics without the overhead of containerization.

**Download and install Node Exporter:**

First, visit the [Node Exporter releases page](https://github.com/prometheus/node_exporter/releases) to find the latest version. At the time of writing, the latest version is 1.10.2.

```bash
# Download Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz

# Extract the archive
tar xvfz node_exporter-1.10.2.linux-amd64.tar.gz

# Move the binary to /usr/local/bin
sudo mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/

# Clean up
rm -rf node_exporter-1.10.2.linux-amd64*
```

**Create a systemd service:**

Create a dedicated user for Node Exporter:

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Create the systemd service file:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Add the following configuration:

```ini title="node_exporter.service"
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --collector.filesystem.mount-points-exclude='^/(sys|proc|dev|host|etc)($$|/)' \
    --collector.netclass.ignored-devices='^(veth.*|br.*|docker.*|virbr.*|lo)$$'

[Install]
WantedBy=multi-user.target
```

**Configuration explained:**

- **collector.filesystem.mount-points-exclude**: Prevents monitoring of virtual filesystems to avoid cluttering metrics with unnecessary data
- **collector.netclass.ignored-devices**: Excludes virtual network interfaces (Docker bridges, veth pairs, etc.) from network metrics

**Start and enable the service:**

```bash
# Reload systemd to recognize the new service
sudo systemctl daemon-reload

# Start Node Exporter
sudo systemctl start node_exporter

# Enable it to start on boot
sudo systemctl enable node_exporter

# Check the status
sudo systemctl status node_exporter
```

Node Exporter is now running on port 9100 and collecting system metrics.

## Setup cAdvisor

cAdvisor runs as a Docker container to monitor other Docker containers. Create a dedicated folder for its configuration:

```bash
mkdir cadvisor
```

Create a `docker-compose.yml` file for cAdvisor:

```bash
nano cadvisor/docker-compose.yml
```

Add the following configuration:
```yaml title="docker-compose.yml"
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    container_name: cadvisor
    privileged: true
    devices:
      - /dev/kmsg:/dev/kmsg
    environment:
      - TZ=Europe/Amsterdam
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /cgroup:/cgroup:ro
    command:
      - '--housekeeping_interval=15s'
      - '--docker_only=true'
    restart: unless-stopped
    networks:
      - backend
networks:
  backend:
    name: backend
```

**Configuration explained:**

- **privileged: true**: Required for cAdvisor to access low-level system information and container metrics
- **devices**: Mounts kernel message device for system logging access
- **volumes**: Mounts necessary host directories to monitor Docker containers and system resources
- **housekeeping_interval**: Sets how often cAdvisor performs cleanup and metric collection (15 seconds)
- **docker_only**: Limits monitoring to Docker containers only, reducing overhead from non-containerized processes

Start cAdvisor:

```bash
docker compose -f cadvisor/docker-compose.yml up -d
```

## Setup Prometheus

To collect the metrics from Node Exporter and cAdvisor, we’ll create a dedicated directory for Prometheus to store its configuration and Docker Compose files.

First, create the Prometheus folder:
```bash
mkdir prometheus
```

Next, create a `docker-compose.yml` file for Prometheus:

```bash
nano prometheus/docker-compose.yml
```

Add the following configuration to the file:

```yaml title="docker-compose.yml"
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    environment:
      - TZ=Europe/Amsterdam
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.size=100GB'
      - '--web.enable-lifecycle'
    restart: unless-stopped
    expose:
      - 9090
    networks:
      - backend
    extra_hosts:
      - "host.docker.internal:host-gateway"

networks:
  backend:
    name: backend

volumes:
    prometheus:
      name: prometheus
```

**Configuration explained:**

- **volumes**: Mounts the Prometheus configuration file and creates persistent storage for time-series data
- **config.file**: Specifies the location of the Prometheus configuration file
- **storage.tsdb.path**: Directory where Prometheus stores its time-series database
- **storage.tsdb.retention.size**: Maximum storage size (100GB) before oldest data is deleted
- **web.enable-lifecycle**: Allows reloading configuration without restarting the container
- **extra_hosts**: Enables Prometheus to reach the host machine using `host.docker.internal`

Prometheus requires a configuration file to define which services to scrape for metrics. Create the configuration file:

```bash
nano prometheus/prometheus.yml
```
Add the following configuration to the file:
```yaml title="prometheus.yml"
global:
  scrape_interval:     15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: 'cadvisor'
    scrape_interval: 10s
    static_configs:
      - targets: ['cadvisor:8080']
  - job_name: 'nodeexporter'
    scrape_interval: 10s
    static_configs:
      - targets: ['host.docker.internal:9100']
```

**Configuration explained:**

- **cadvisor:8080**: Connects to cAdvisor container via Docker's backend network
- **host.docker.internal:9100**: Allows the Prometheus container to reach Node Exporter running on the host system. The `extra_hosts` configuration in the Prometheus docker-compose.yml makes this possible.

Now that you have configured Prometheus, you can start it with the following command:

```bash
docker compose -f prometheus/docker-compose.yml up -d
```

This command starts the Prometheus container, which will begin collecting metrics from both Node Exporter and cAdvisor, providing comprehensive monitoring for your systems and containers.

## Setup Grafana

To finalize your monitoring setup, we’ll create a directory for Grafana to store its Docker Compose and configuration files.

First, create the Grafana folder:
```bash
mkdir grafana
```

Next, create a `docker-compose.yml` file for Grafana:

```bash
nano grafana/docker-compose.yml
```
Add the following configuration to the file:
```yaml title="docker-compose.yml"
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    environment:
      - TZ=Europe/Amsterdam
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped
    ports:
      - 3000:3000
    networks:
      - backend

networks:
  backend:
    name: backend

volumes:
    grafana_data:
      name: grafana_data
```

Now you can start Grafana by running:

```bash
docker compose -f grafana/docker-compose.yml up -d
```

Once Grafana is running, open your browser and navigate to `http://<HOST_IP>:3000` (replace `<HOST_IP>` with your server's IP address).

- Default login credentials:
    - Username: **admin**
    - Password: **admin**

### Datasource

To visualize the data collected by Prometheus, you need to add it as a data source in Grafana:

1.  Click **Connections** in the left-side menu.
2.  Search for **Prometheus**
3.  Click **Add new Datasource**
4.  Enter the name **prometheus**
5.  Fill in the Prometheus server URL `http://prometheus:9090`

Dashboards

To see all the metrics we need to dashboards. You can make your own dashboards or use mine as a starter:

*   [NodeExporter]
*   [cAdvisor]

## Conclusion

Congratulations! You have successfully set up host and container monitoring with Prometheus and Grafana. Your monitoring system is now capable of visualizing the metrics from your applications and self-hosting.

[nodeexporter]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/node_expoter
[cadvisor]: https://github.com/svenvg93/Grafana-Dashboard/tree/master/cadvisor
