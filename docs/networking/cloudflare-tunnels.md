---
title: Cloudflare Tunnels
description: Securely exposing your services with Cloudflare Tunnels.
tags:
- cloudflare
- docker
- zero-trust
---


In the world of self-hosting, secure and reliable server access is crucial. Cloudflare Zero Trust offers a solution through Cloudflare Tunnels, allowing secure access to self-hosted services without opening ports or changing firewall settings. By creating an outbound-only connection to Cloudflare, traffic remains encrypted and routed through its global network, enhancing security and performance while protecting your server from direct attacks.

??? info "Prerequisites"
    - A **Cloudflare account** with a domain added.
    - **Docker** and **Docker Compose** installed on the host.

## Create Cloudflare Tunnel

### Docker Compose

Create a `docker-compose.yml` file to define the Cloudflare Tunnel service:

```yaml title="docker-compose.yml" hl_lines="7"
services:
  cloudflared:
    image: cloudflare/cloudflared
    container_name: cloudflared
    environment:
      - TZ=[TIMEZONE]
      - TUNNEL_TOKEN=${TOKEN}
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    networks:
      - cloudflared
networks:
  cloudflared:
    name: cloudflared
```

### Obtain Your Tunnel Token

Create a tunnel in your Cloudflare account:

- Go to [Cloudflare Dashboard]
- Go to **Zero Trust**
- Click **Networks** -> **Tunnels**
- Click on **Add a Tunnel**
- Select **Cloudflared** and click **Next**
- Give your tunnel a name
- Click **Save Tunnel**
- Choose **Docker** as the installation method
- Copy the command with the copy button

Create a `.env` file in the same directory as your `docker-compose.yml`:

``` title=".env"
TOKEN=<Your tunnel token>
```

Replace `<Your tunnel token>` with the string found after `--token` in the command you copied from Cloudflare.

!!! warning "Keep your token secure"
    The tunnel token grants full access to your tunnel. Never commit the `.env` file to version control. Add it to your `.gitignore`.

Start the tunnel:

```bash
docker compose up -d
```

If everything is correct, your tunnel will show as connected within a few seconds.

## Add a Service

To verify that everything is working, start a simple test service using the [whoami application]:

```yaml title="docker-compose.yml"
services:
  whoami:
    container_name: simple-service
    image: traefik/whoami
    networks:
      - cloudflared
networks:
  cloudflared:
    name: cloudflared
```

```bash
docker compose up -d
```

Now go back to the Cloudflare dashboard and click **Next**. Fill in the following fields:

- **Subdomain:** the subdomain you want to use (e.g., `test`)
- **Domain:** select your domain name from the list
- **Type:** choose `HTTP`
- **URL:** the IP address and port of the application, or the container name if it's on the same Docker network as the tunnel

Click **Save Tunnel** to complete the setup.

??? tip "Verify your tunnel is working"
    Run `nslookup` and `traceroute` for your configured domain. You'll notice all traffic routes through Cloudflare and your server's IP address is not visible, confirming the tunnel is masking your origin.

[cloudflare dashboard]: https://dash.cloudflare.com/
[whoami application]: https://github.com/traefik/whoami
