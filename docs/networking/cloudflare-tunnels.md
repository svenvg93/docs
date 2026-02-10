---
title: Cloudflare Tunnels
description: Securely exposing your services with Cloudflare Tunnels.
tags:
- cloudflare
- docker
- zero-trust
---

Cloudflare Tunnels create an outbound-only connection from your server to Cloudflare's network, allowing you to expose self-hosted services without opening ports or modifying firewall rules. All traffic is encrypted and routed through Cloudflare, keeping your server's IP address hidden.

??? info "Prerequisites"
    - A Cloudflare account with a domain added
    - Docker and Docker Compose installed on the host

## Obtain Tunnel Token

Create a tunnel in the [Cloudflare Dashboard]:

1. Navigate to **Zero Trust** > **Networks** > **Tunnels**
2. Click **Add a Tunnel**
3. Select **Cloudflared** and click **Next**
4. Give your tunnel a name and click **Save Tunnel**
5. Choose **Docker** as the installation method
6. Copy the token from the provided command (the string after `--token`)

Create a `.env` file to store your tunnel token:

```bash title=".env"
TOKEN=eyJhIjoiY... # (1)!
```

1. :material-key: **Tunnel Token** - Replace with the token you copied from the Cloudflare dashboard.

!!! warning "Keep your token secure"
    The tunnel token grants full access to your tunnel. Never commit the `.env` file to version control. Add it to your `.gitignore`.

## Create Tunnel

Create a `docker-compose.yml` file to define the Cloudflare Tunnel service:

```yaml title="docker-compose.yml"
services:
  cloudflared:
    image: cloudflare/cloudflared
    container_name: cloudflared
    environment:
      - TZ=Europe/Amsterdam # (1)!
      - TUNNEL_TOKEN=${TOKEN} # (2)!
    restart: unless-stopped
    command: tunnel --no-autoupdate run # (3)!
    networks:
      - cloudflared
networks:
  cloudflared:
    name: cloudflared # (4)!
```

1. :material-clock-outline: **Timezone** - Your local timezone
2. :material-key: **Token** - References the token from your `.env` file
3. :material-update: **Command** - Runs the tunnel and disables auto-updates (managed via Docker image instead)
4. :material-lan: **Network** - Named network that other services can join to be accessible through the tunnel

Start the tunnel:

```bash
docker compose up -d
```

## Add a Service

To route traffic through the tunnel, go back to the Cloudflare dashboard and configure a public hostname:

1. Click **Next** on the tunnel setup page
2. Fill in the following fields:
    - **Subdomain**: the subdomain you want to use (e.g., `test`)
    - **Domain**: select your domain from the list
    - **Type**: choose `HTTP`
    - **URL**: the container name or IP address of the application
3. Click **Save Tunnel**

!!! note "Docker networking"
    Services must be on the same Docker network as the tunnel (`cloudflared`). Add the network to any service you want to expose:

    ```yaml
    networks:
      - cloudflared
    ```

## Verification

Verify the tunnel is working correctly:

1. Check the tunnel status in the Cloudflare dashboard — it should show as **Connected**
2. Navigate to your configured subdomain (e.g., `https://test.example.com`) and confirm the service loads
3. Run `nslookup` on your domain to confirm traffic routes through Cloudflare and your server's IP is not exposed

[Cloudflare Dashboard]: https://dash.cloudflare.com/
