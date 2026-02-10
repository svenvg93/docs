---
title: Unifi Controller via Traefik 
description: Proxy Unifi Controller through Traefik using file-based configuration
tags:
- unifi
- traefik
---

Some services, like the Unifi Controller, use self-signed HTTPS certificates and don't work with Traefik's label-based auto-discovery. In these cases, use a file-based configuration. Traefik connects to the controller's HTTPS backend, bypasses the self-signed certificate, and presents a valid Let's Encrypt certificate to your browser.

??? info "Prerequisites"
    - Traefik installed and configured (see the [Traefik Setup & Configuration][traefik-setup] guide)
    - Unifi Controller running and accessible on your network

## Traefik Configuration

### Enable File Provider

First, ensure Traefik can read dynamic configuration files. Add the following to your Traefik container in `docker-compose.yml`:

**Command arguments:**
```yaml title="docker-compose.yml"
command:
  - "--providers.file.directory=/etc/traefik/dynamic" # (1)!
  - "--providers.file.watch=true" # (2)!
  # ... your other Traefik arguments
```

1. Points Traefik to dynamic config files for non-Docker services
2. Enables automatic reload on config changes

**Volume mounts:**
```yaml title="docker-compose.yml" hl_lines="4"
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - traefik:/certs
  - ./config:/etc/traefik/dynamic:ro # (1)!
```

1. Maps the local config directory to Traefik’s dynamic path (read-only)

### Create Dynamic Configuration File

Create a new file at `./config/unifi.yml` with the following configuration:

```yaml title="unifi.yml" hl_lines="4 5 7 9 15 16 20"
http:
  routers:
    unifi:
      rule: "Host(`unifi.lab.example.com`)" # (1)!
      service: unifi-service # (2)!
      entryPoints: 
        - websecure # (3)!
      tls: 
        certResolver: le  # (4)!

  services:
    unifi-service:
      loadBalancer:
        servers:
          - url: "https://192.168.1.1" # (5)!
        serversTransport: unifi-transport # (6)!

  serversTransports: 
    unifi-transport:
      insecureSkipVerify: true # (7)!
```

1. The domain that will route to your Unifi Controller
2. References the service definition below for backend routing
3. Uses the `websecure` entry point for HTTPS traffic
4. Uses Let's Encrypt (`le`) to provide valid SSL certificates to clients
5. The IP address and protocol of your Unifi Controller (replace `192.168.1.1` with your controller's IP)
6. References the custom transport configuration that handles SSL verification
7. Set to `true` because Unifi Controller uses self-signed certificates internally.

## Apply Configuration

Re-create the Traefik container to apply the changes:

```bash
docker compose -f traefik/docker-compose.yml up -d --force-recreate
```

## Verification

After restarting, verify the configuration is working:

1. Check Traefik logs for errors loading the file provider:

```bash
docker logs traefik | grep -i "unifi"
```

2. Navigate to `https://unifi.lab.example.com` in your browser and confirm:
    - No SSL certificate warnings appear
    - The Unifi Controller login page loads correctly

[traefik-setup]: ../../reverse-proxy/traefik/install-setup
