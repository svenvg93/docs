---
title: Unifi Controller via Traefik 
categories:
- networking
date: 2025-12-14
description: Proxy Unifi Controller through Traefik using file-based configuration
draft: false
tags:
- unifi
- traefik
---


While Traefik excels at auto-discovering Docker containers through labels, some services like the Unifi Controller require a different approach. The Unifi Controller uses self-signed certificates and runs on HTTPS, making it a perfect candidate for Traefik's file-based configuration with `insecureSkipVerify`.

## Key Components

1. Traefik - Reverse proxy that handles SSL termination and routes traffic to the Unifi Controller.
2. Unifi Controller - Network management software running with self-signed HTTPS certificates.
3. File Provider - Traefik's dynamic configuration system for non-Docker services.
4. Let's Encrypt - Provides valid SSL certificates for external access without browser warnings.

!!! info "Prerequisites"
    - Traefik installed and configured (see the [Traefik Setup & Configuration][traefik-setup] guide)
    - Unifi Controller running and accessible on your network

## Traefik Configuration

### Enable File Provider

First, ensure Traefik can read dynamic configuration files. Add the following to your Traefik container in `docker-compose.yml`:

**Command arguments:**
```yaml
command:
  - "--providers.file.directory=/etc/traefik/dynamic"
  - "--providers.file.watch=true"
  # ... your other Traefik arguments
```

**Volume mounts:**
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - traefik:/certs
  - ./config:/etc/traefik/dynamic:ro
```

**Configuration explained:**

- **providers.file.directory**: Tells Traefik where to find dynamic configuration files for non-Docker services
- **providers.file.watch**: Enables automatic reloading when configuration files change (no container restart needed)
- **./config volume**: Maps the local `config` directory to Traefik's dynamic configuration path in read-only mode

### Create Dynamic Configuration File

Create a new file at `./config/unifi.yml` with the following configuration:

```yaml
http:
  routers:
    unifi:
      rule: "Host(`unifi.lab.example.com`)"
      service: unifi-service
      entryPoints:
        - websecure
      tls:
        certResolver: le

  services:
    unifi-service:
      loadBalancer:
        servers:
          - url: "https://192.168.1.1"
        serversTransport: unifi-transport

  serversTransports:
    unifi-transport:
      insecureSkipVerify: true
```

**Configuration explained:**

**Router Configuration:**

- **rule**: The domain that will route to your Unifi Controller (replace `unifi.lab.example.com` with your domain)
- **service**: References the service definition below for backend routing
- **entryPoints**: Uses the `websecure` entry point for HTTPS traffic
- **tls.certResolver**: Uses Let's Encrypt (`le`) to provide valid SSL certificates to clients

**Service Configuration:**

- **url**: The IP address and protocol of your Unifi Controller (replace `192.168.1.1` with your controller's IP)
- **serversTransport**: References the custom transport configuration that handles SSL verification

**Servers Transport:**

- **insecureSkipVerify**: Set to `true` because Unifi Controller uses self-signed certificates internally. This allows Traefik to connect to the backend without SSL verification errors.

## How It Works

When you navigate to `unifi.lab.example.com`:

1. **DNS Resolution**: Your DNS resolves the domain to your Traefik server
2. **SSL Termination**: Traefik presents a valid SSL certificate to your browser
3. **Backend Connection**: Traefik connects to the Unifi Controller's HTTPS interface at `192.168.1.1`
4. **Certificate Bypass**: The `insecureSkipVerify` setting allows Traefik to accept Unifi's self-signed certificate
5. **Secure Access**: You access your Unifi Controller with a valid SSL certificate, no browser warnings

This file-based approach gives you full control over services that can't use Docker labels, while still benefiting from Traefik's reverse proxy capabilities and automatic SSL certificate management.

[traefik-setup]: ../../reverse-proxy/traefik/install-setup
