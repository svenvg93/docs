---
title: Setup & Configuration
description: Learn to deploy and configure Traefik for seamless traffic management in your homelab.
tags:
- traefik
- cloudflare
- lets encrypt
---


## What is Traefik

Traefik is an open-source reverse proxy and load balancer that works well with Docker, automatically detecting services and securing connections with SSL. It adapts in real-time, making it ideal for dynamic homelab setups.

This guide shows how to set up Traefik in Docker, enable automatic Let’s Encrypt SSL, and test it with a simple service.

## Setup Traefik with Docker

### Docker

Create the directory and `docker-compose.yml`

```bash
mkdir traefik
nano traefik/docker-compose.yml
```

Add the following configuration to the file:

```yaml title="docker-compose.yml"
services:
  traefik:
    image: traefik:3.6.7
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    environment:
      - TZ=Europe/Amsterdam # (1)!
    env_file:
      - .env
    command:
      - "--api.insecure=true"
      - "--api=true"
      - "--api.dashboard=true"
      - "--ping=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik"
      - "--entryPoints.web.address=:80"
      - "--entryPoints.websecure.address=:443"
      - "--entryPoints.websecure.http.tls=true"
      - "--entryPoints.web.http.redirections.entryPoint.to=websecure"
      - "--entryPoints.web.http.redirections.entryPoint.scheme=https"
      - "--certificatesresolvers.le.acme.dnschallenge=true"
      - "--certificatesresolvers.le.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.le.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.le.acme.dnschallenge.delaybeforecheck=60s"      
      - "--certificatesresolvers.le.acme.storage=/certs/acme.json"
      - "--log.level=INFO"
    networks:
      - traefik
    ports:
      - 80:80 # Web entryPoints
      - 443:443 # Websecure entryPoints
      - 8080:8080 # Traefik Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_data:/certs # (2)!
    healthcheck:
      test: wget --quiet --tries=1 --spider  http://127.0.0.1:8080/ping || exit 1
      interval: 5s
      timeout: 1s
      retries: 3
      start_period: 10s

volumes:
  traefik_data:
    name: traefik_data

networks:
  traefik:
    name: traefik
```

1. Change to your local timezone (e.g., `America/New_York`, `UTC`)
2. Docker volume to store cerficates


## Cloudflare API

This guide uses Cloudflare as the DNS provider, but you can follow similar steps for other providers (see the [lego docs] for supported providers).

To use the DNS-01 challenge with Cloudflare:

1. **Create an API token** in your Cloudflare dashboard with `DNS:Edit` permissions. Traefik will use this token to authenticate with Cloudflare for automatic SSL certificate management.

2. **Store your credentials** securely by creating a `.env` file in the same directory as your `docker-compose.yml`:

```bash
nano .env
```

Add the following content to the `.env` file, replacing the placeholders with your actual Cloudflare email and API token:

```bash title=".env"
CF_API_EMAIL=<Your cloudflare email>
CF_DNS_API_TOKEN=<Your API Token>
DOMAIN=<your-domain>
ACME_EMAIL=<Your-mail>
```
## Start Traefik

Run the command below to start the container.

```bash
docker compose -f traefik/docker-compose.yml up -d
```

Access the dashboard at `http://<server-ip>:8080` (replace `<server-ip>` with your Traefik server's IP address).

## Add service

To verify Traefik is working correctly, we’ll set up a simple service using Traefik’s whoami application. This service provides basic HTTP responses displaying browser and OS information, which makes it ideal for testing.

Make a new directory called whoami to organize the service files.

```bash
mkdir whoami
nano whoami/docker-compose.yml
```

Add the following configuration to the file:

```yaml  title="docker-compose.yml"
services:
  whoami:
    container_name: simple-service
    image: traefik/whoami
    labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami.rule=Host(`whoami.${DOMAIN}`)"
        - "traefik.http.routers.whoami.entrypoints=websecure"
        - "traefik.http.routers.whoami.tls=true"
        - "traefik.http.routers.whoami.tls.certresolver=le"
        - "traefik.http.services.whoami.loadbalancer.server.port=80"
    networks:
      - traefik

networks:
  traefik:
    name: traefik
```

### Setting Up DNS and Testing

1. **Configure DNS**: Point `whoami.your-domain.com` to your server's IP address in your DNS settings.
2. **Verify DNS propagation**: Use `nslookup` or an online DNS checker to confirm the domain resolves to your server.
3. **Start the service**:
   ```bash
   docker compose -f whoami/docker-compose.yml up -d
   ```
4. **Test the setup**: Open your browser and navigate to `https://whoami.your-domain.com`. You should see the whoami response page with a valid SSL certificate.
5. **Clean up** (optional): Once verified, you can remove the test service:
   ```bash
   docker compose -f whoami/docker-compose.yml down
   ```

[lego docs]: https://go-acme.github.io/lego/dns/