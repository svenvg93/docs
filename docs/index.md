---
title: Welcome
description: Comprehensive guides for self-hosting, infrastructure, automation, and monitoring
icon: lucide/home
--- 

# Welcome to Tech Documentation

This documentation site contains comprehensive guides for setting up and managing self-hosted infrastructure, automation tools, monitoring systems, and more.

## What You'll Find Here

### :material-server-network: Infrastructure

Learn how to set up and manage your infrastructure:

- **[Reverse Proxy]**: Traefik setup, SSL certificates, and advanced configurations
- **[Networking]**: Cloudflare Tunnels, Tailscale, VPNs, and DNS management
- **[Routers]**: MikroTik, VyOS, and travel router configurations

### :material-chart-line: Monitoring & Observability

Build comprehensive monitoring solutions:

- **[System Monitoring]**: Prometheus and Grafana for hosts and containers
- **[Log Management]**: Centralized logging with Loki
- **[Network Monitoring]**: Track your network infrastructure

### :material-shield-lock: Security

Implement security best practices:

- **[Authentication]**: Single Sign-On with Authentik
- **[Zero Trust Access]**: Secure your infrastructure
- **[SSL/TLS]**: Certificate management

### :material-robot: Automation

Automate your infrastructure:

- **[Ansible]**: Configuration management and automation
- **[CI/CD]**: Automated dependency updates
- **[GitOps]**: Infrastructure as Code

### :material-tools: Tools & Utilities

Explore useful tools:

- **[Documentation Platforms]**: Static site generators (Zensical, Hugo, Jekyll, MkDocs)
- **[Notifications]**: Self-hosted notification systems
- **[Analytics]**: Privacy-focused web analytics

## Getting Started

If you're new to self-hosting, we recommend starting with:

1. [Setting up Traefik as a reverse proxy]
2. [Implementing monitoring with Prometheus]
3. [Securing access with Authentik SSO]

[reverse-proxy]: infrastructure/reverse-proxy/index.md
[networking]: infrastructure/networking/index.md
[routers]: infrastructure/routers/index.md
[system-monitoring]: monitoring/host-container-monitoring-with-prometheus.md
[log-management]: infrastructure/monitoring/system-logs-loki.md
[network-monitoring]: infrastructure/networking/monitoring-tailscale-clients-with-prometheus.md
[authentication]: security/selfhost-a-single-sign-on-mfa-with-authentik.md
[zero-trust-access]: security/secure-your-cloudflare-tunnel-with-authentik.md
[ssltls]: infrastructure/reverse-proxy/wildcard-ssl-certificates-with-traefik-and-unifi-local-dns.md
[ansible]: automation/ansible/index.md
[cicd]: automation/automating-dependabot-for-docker-compose.md
[gitops]: infrastructure/managing-portainer-stacks-using-gitops.md
[documentation-platforms]: tools/documentation/index.md
[notifications]: tools/notifications/setup-ntfy-for-selfhosted-notifications.md
[analytics]: tools/analytics/selfhost-your-website-analytics-with-umami.md
[setting-up-traefik-as-a-reverse-proxy]: infrastructure/reverse-proxy/traefik-essentials-reverse-proxy-with-docker-lets-encrypt.md
[implementing-monitoring-with-prometheus]: monitoring/host-container-monitoring-with-prometheus.md
[securing-access-with-authentik-sso]: security/selfhost-a-single-sign-on-mfa-with-authentik.md
