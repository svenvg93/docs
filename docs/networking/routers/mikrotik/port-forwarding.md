---
title: Port Forwarding
description: Configure port forwarding on a MikroTik router to expose services from your homelab to the internet.
tags:
- mikrotik
- firewall
- port-forwarding
---

Port forwarding allows you to make services running on your internal network accessible from the internet. This is useful for hosting web servers, game servers, or other services in your homelab. Below we'll walk you through the steps to configure port forwarding on your MikroTik router.

!!! tip
    Replace the placeholder variables on the highlighted lines with values that match your environment.

## Firewall Rule

If you followed the [Initial Setup](install-setup.md) guide, you already have a firewall rule that allows forwarded traffic for destination NAT. If not, add the following rule to allow traffic that has been destination-NATed to pass through the firewall:

```bash
/ip firewall filter
add action=accept chain=forward comment="port forwarding" connection-nat-state=dstnat
```

!!! warning
    Make sure this rule is placed **before** any drop rules in the forward chain, otherwise the forwarded traffic will be blocked.

## NAT Rule

Create a destination NAT rule to redirect incoming traffic on a specific port to an internal server. This tells the router to forward traffic arriving on the WAN interface to a device on your local network.

```bash hl_lines="2"
/ip firewall nat
add action=dst-nat chain=dstnat comment="Port forward" dst-port=80 protocol=tcp to-addresses=172.16.10.3 to-ports=80
```

| Parameter       | Description                                      |
|-----------------|--------------------------------------------------|
| `dst-port`      | The external port to listen on                   |
| `protocol`      | The protocol to match (`tcp` or `udp`)           |
| `to-addresses`  | The internal IP address of the destination server|
| `to-ports`      | The internal port on the destination server       |

!!! tip "Multiple ports"
    To forward multiple ports for the same server, create additional NAT rules with different `dst-port` and `to-ports` values. You can also specify a port range using a dash, for example `dst-port=8000-8010`.
