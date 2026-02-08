---
title: Remote Packet Capture
description: A guide to setting up Wireshark for secure remote packet capture via SSH.
tags:
- Wireshark
- capture
- network
---


This guide shows how to set up **Wireshark** to remotely capture traffic via **SSH** using `tcpdump`.

??? info "Prerequisites"
    - **Wireshark** installed on your local machine.
    - **SSH access** to the remote host.
    - **tcpdump** installed on the remote host.

## Setup Remote Host

Wireshark runs tcpdump non-interactively over SSH, so `sudo` will fail if it requires a password prompt. Grant tcpdump the capture capability directly on the remote host.

```bash
sudo setcap cap_net_raw+ep $(which tcpdump)
```

## Setup Wireshark

1. Go to: `Capture` → `Options` → `Manage Interfaces`
2. Click on the gear icon next to **SSH Remote Capture**
3. Set the interface details:
   - **Remote SSH Server Address:** `ip-address of the server`
   - **Remote SSH Server port:** `22`
4. In the **Authentication** tab, fill in:
   - **Remote SSH Server Username**
   - **Remote SSH Server Password** or **Path to SSH Private Key** :lucide-info:{ title="~/.ssh/id_rsa" .env-tooltip }

5. In the **Capture** tab, configure what to capture:

    - **Remote Interface:** the network interface on the remote host (e.g., `eth0`).
    - **Remote Capture Filter:** a BPF filter to limit captured traffic :lucide-info:{ title="Example: not port 22" .env-tooltip }
    
6. Save and **Start** the capture.
