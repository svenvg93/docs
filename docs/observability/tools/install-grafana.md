---
title: Grafana
description: How to install Grafana for metrics and logs visualization
tags:
    - docker
    - grafana
    - dashboards
---

# Setup Grafana

Grafana is an open-source visualization and analytics platform. It allows you to query, visualize, and alert on metrics and logs from various data sources including Prometheus and Loki.

??? info "Prerequisites"

    Before installing Grafana, ensure you have:

    - **Docker** and **Docker Compose** installed on your system

??? question "Why Grafana?"

    Grafana is the leading visualization platform for observability:

    - **Multi-datasource**: Connect to Prometheus, Loki, InfluxDB, and many more
    - **Rich dashboards**: Create interactive dashboards with graphs, tables, and alerts
    - **Alerting**: Built-in alerting with multiple notification channels
    - **Community dashboards**: Thousands of pre-built dashboards available
    - **Explore mode**: Ad-hoc querying for logs and metrics

## Installation

Create a directory for Grafana:

```bash
mkdir grafana
```

Create the Docker Compose file:

```bash
nano grafana/docker-compose.yml
```

```yaml title="docker-compose.yml"
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    hostname: ${HOSTNAME} # (1)!
    environment:
      - TZ=Europe/Amsterdam # (2)!
      - GF_SECURITY_ADMIN_PASSWORD=admin # (3)!
    volumes:
      - grafana_data:/var/lib/grafana # (4)!
    restart: unless-stopped
    ports:
      - 3000:3000 # (5)!
    networks:
      - backend

networks:
  backend:
    name: backend

volumes:
  grafana_data:
    name: grafana_data
```

1. :material-server: **Hostname** - Inherits the host system's hostname for identification
2. :material-clock-outline: **Timezone** - Change to your local timezone (e.g., `America/New_York`, `UTC`)
3. :material-lock: **Admin Password** - Change this or remove to set password on first login
4. :material-database: **Data Volume** - Persistent storage for dashboards, users, and settings
5. :material-web: **Web UI Port** - Access Grafana at port 3000

??? tip "Additional environment variables"
    Grafana can be configured via environment variables:

    ```yaml
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secure_password
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=https://grafana.example.com
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.example.com:587
    ```

    See the [Grafana configuration docs] for all options.

## Starting Grafana

Start the Grafana service:

```bash
docker compose -f grafana/docker-compose.yml up -d
```

## Verification

After starting Grafana, verify the installation:

**Check container status:**

```bash
docker ps | grep grafana
```

**View Grafana logs:**

```bash
docker logs grafana
```

**Access the Web UI:**

Open `http://<HOST_IP>:3000` in your browser.

- Default username: **admin**
- Default password: **admin** (or value of `GF_SECURITY_ADMIN_PASSWORD`)

You'll be prompted to change the password on first login.

## Adding Datasources

### Prometheus

To visualize metrics from Prometheus:

1. Click **Connections** in the left-side menu
2. Search for **Prometheus**
3. Click **Add new Datasource**
4. Enter the name `prometheus`
5. Set the URL to `http://prometheus:9090`
6. Click **Save & Test**

### Loki

To visualize logs from Loki:

1. Click **Connections** in the left-side menu
2. Search for **Loki**
3. Click **Add new Datasource**
4. Enter the name `loki`
5. Set the URL to `http://loki:3100`
6. Click **Save & Test**

[Grafana configuration docs]: https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/
