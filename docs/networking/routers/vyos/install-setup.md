---
title: Initial Setup
description: Setting up a VyOS router for your homelab gives you enterprise-grade networking with open-source flexibility.
tags:
- vyos
- firewall
---

Setting up a [VyOS] router in your homelab gives you enterprise-grade networking with open-source flexibility. Running on standard amd64 hardware, VyOS delivers powerful routing and firewall features—far beyond what typical consumer routers offer.

## Installation

??? warning "Production use"
    VyOS rolling release images are built from the latest development code, incorporating the newest changes from maintainers and community contributors. While they receive automated testing to ensure they boot and load configurations, they may include experimental features, bugs, and compatibility issues. As a result, they are not recommended for production use.

After [downloading] the image, boot from USB, VM, or PXE. Log in with `vyos`/`vyos`, run `install image`, follow the wizard to set the root password, then remove the media and reboot.

!!! tip
    Replace the placeholder variables in brackets with values that match your environment.

## Operational modes

VyOS has two main operational modes: Operational Mode and Configuration Mode. Understanding these modes is key to managing and configuring the system effectively.

??? note "Operational Mode" 
    The default mode when you log in. Used for monitoring, troubleshooting, and running system commands. Commands in this mode do not change the system's configuration.
        
??? note "Configuration Mode" 
    Used to modify the system's settings.

Enter configuration mode to begin the initial setup:

```bash
configure
```

!!! info "commit and save"
    Throughout this guide you'll see `commit; save` after each configuration block. The `commit` command applies changes to the running configuration, while `save` writes them to disk. If you only `commit` without `save`, your changes will be lost on reboot.

## LAN

Configure the LAN ports to establish a network connection for all your devices. This will ensure that both your homelab and internet access are set up properly, providing seamless connectivity throughout your network.

### Bridge Interface

Create a bridge interface to combine all the ports into a single network. This will enable seamless communication between all your devices on the same network.

!!! tip "LAN Subnet"
    This guide uses `192.168.1.0/24` as the LAN subnet. Change it everywhere if you modify it.

```bash
set interfaces bridge br0 
set interfaces bridge br0 description LAN-bridge
set interfaces bridge br0 address 192.168.1.1/24
set interfaces bridge br0 member interface eth0
commit; save
```

Repeat the `member interface` command for every interface you want to be part of the bridge, replacing `eth0` with the appropriate interface name (e.g., `eth2`, `eth3`).

You can check the bridge with the command `run show bridge br0`


```bash
admin@BR01:~$ run show interfaces bridge 
Codes: S - State, L - Link, u - Up, D - Down, A - Admin Down
Interface        IP Address                        S/L  Description
---------        ----------                        ---  -----------
br0              192.168.1.1/24                    u/u 
```

!!! tip
    When in Configuration Mode, you normally can't run operational commands like `show`. However, you can use `run` before the command to execute it without leaving Configuration Mode.

### DHCP

Set up a DHCP server to automatically assign IP addresses to devices connected to your network.

```bash
set service dhcp-server shared-network-name LAN authoritative
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 lease 86400
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 option default-router 192.168.1.1
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 option name-server 192.168.1.1
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 range 0 start 192.168.1.100
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 range 0 stop 192.168.1.200
set service dhcp-server shared-network-name LAN subnet 192.168.1.0/24 subnet-id 1
commit; save
```

To view active leases from connected clients, use the command: `run show dhcp server leases`

```bash
admin@BR01:~$ run show dhcp server leases
IP Address     MAC address        State    Lease start                Lease expiration           Remaining    Pool    Hostname     Origin
-------------  -----------------  -------  -------------------------  -------------------------  -----------  ------  -----------  --------
192.168.1.100  bc:24:11:82:b2:20  active   2025-03-19 17:31:03+00:00  2025-03-20 17:31:03+00:00  23:41:55     LAN     ubuntu-test  local
192.168.1.101  bc:24:11:89:c8:77  active   2025-03-19 17:36:13+00:00  2025-03-20 17:36:13+00:00  23:47:05     LAN     ubuntu-test  local
```

??? tip "SSH Access"
    To simplify the configuration, you can now enable SSH, allowing you to copy and paste the remaining steps. 
    Refer to the [SSH] section for instructions on how to do this.

### DNS

By default, VyOS doesn't function as a DNS proxy. Enable DNS forwarding from client devices to your upstream DNS servers:

```bash hl_lines="4"
set service dns forwarding allow-from '192.168.1.0/24' # (1)!
set service dns forwarding listen-address '192.168.1.1' # (2)!
set service dns forwarding system # (3)!
set system name-server [DNS_SERVER] # (4)!
commit; save
```

1. :material-shield-check: **Allow From** - Allows DNS requests from devices in the 192.168.1.0/24 subnet
2. :material-server: **Listen Address** - Sets your VyOS router (192.168.1.1) as the listening address for DNS requests
3. :material-dns: **System DNS** - Enables system-wide DNS forwarding
4. :material-arrow-right: **Upstream DNS** - Forwards requests to your specified upstream DNS server

## WAN

Configure the WAN interface to obtain an IP address from your ISP. Select the appropriate method for your connection type.

=== "DHCP with VLAN"
    ```bash hl_lines="1 2"
    set interfaces ethernet [WAN_INTERFACE] vif [VLAN_ID] address dhcp
    set interfaces ethernet [WAN_INTERFACE] vif [VLAN_ID] description Internet
    commit; save
    ```

=== "DHCP"
    ```bash
    set interfaces ethernet eth1 address dhcp
    set interfaces ethernet eth1 description Internet
    ```

=== "PPPoE with VLAN"
    ```bash hl_lines="1 2 3 4"
    set interfaces ethernet [WAN_INTERFACE] vif [VLAN_ID] description Internet
    set interfaces pppoe pppoe0 authentication username [USERNAME]
    set interfaces pppoe pppoe0 authentication password [PASSWORD]
    set interfaces pppoe pppoe0 source-interface [WAN_INTERFACE].[VLAN_ID]
    set interfaces pppoe pppoe0 default-route auto
    set interfaces pppoe pppoe0 mtu 1492
    set interfaces pppoe pppoe0 description Internet
    commit; save
    ```

=== "PPPoE"
    ```bash hl_lines="1 2 3"
    set interfaces pppoe pppoe0 authentication username [USERNAME]
    set interfaces pppoe pppoe0 authentication password [PASSWORD]
    set interfaces pppoe pppoe0 source-interface [WAN_INTERFACE]
    set interfaces pppoe pppoe0 default-route auto
    set interfaces pppoe pppoe0 mtu 1492
    set interfaces pppoe pppoe0 description Internet
    commit; save
    ```

=== "Static IP"
    ```bash hl_lines="1 2 4 5 6"
    set interfaces ethernet [WAN_INTERFACE] description Internet
    set interfaces ethernet [WAN_INTERFACE] address [IP_ADDRESS]/[PREFIX_LENGTH]
    set interfaces ethernet [WAN_INTERFACE] mtu 1500
    set protocols static route 0.0.0.0/0 next-hop [GATEWAY]
    set system name-server [DNS_SERVER]
    set system name-server [DNS_SERVER_2]
    commit; save
    ``` 

Verify the routing table is correct. There should be at least a 0.0.0.0 default route:

```bash
admin@BR01# run show ip route
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

S>* 0.0.0.0/0 [210/0] via 85.146.118.xx, eth1.300, weight 1, 00:00:16
C>* 85.146.118.xx/25 is directly connected, eth1.300, weight 1, 00:00:17
K * 85.146.118.xx/25 [0/0] is directly connected, eth1.300, weight 1, 00:00:17
L>* 85.146.118.xx/32 is directly connected, eth1.300, weight 1, 00:00:17
C>* 192.168.1.0/24 is directly connected, br0, weight 1, 00:07:15
L>* 192.168.1.1/32 is directly connected, br0, weight 1, 00:07:15
```

## NAT

Set up a NAT rule to translate all outgoing traffic from your local network to your public IP address. This enables devices in your homelab to access the internet using the router's public IP.

```bash hl_lines="2"
set nat source rule 10 description 'Enable NAT on WAN'
set nat source rule 10 outbound-interface name [WAN_INTERFACE]
set nat source rule 10 translation address 'masquerade'
commit; save
```

## Firewall

In VyOS (and most Netfilter/iptables-based firewalls), traffic is filtered through three main chains: INPUT, OUTPUT, and FORWARD. Knowing these chains is key to setting effective firewall rules.

### Input Chain

This controls incoming traffic destined for the VyOS router itself. For example, SSH access to the router or web management interfaces would be filtered by the INPUT chain.

```bash hl_lines="7"
set firewall ipv4 input filter rule 5 action 'drop'
set firewall ipv4 input filter rule 5 state 'invalid'
set firewall ipv4 input filter rule 5 description 'Drop invalid state packets'
set firewall ipv4 input filter rule 10 action 'accept'
set firewall ipv4 input filter rule 10 state 'established'
set firewall ipv4 input filter rule 10 state 'related'
set firewall ipv4 input filter rule 10 inbound-interface name [WAN_INTERFACE]
set firewall ipv4 input filter rule 10 description 'Allow Return traffic destined to the router'
set firewall ipv4 input filter rule 1000 action 'accept'
set firewall ipv4 input filter rule 1000 description 'Allow all traffic from LAN interface'
set firewall ipv4 input filter rule 1000 inbound-interface name br0
set firewall ipv4 input filter default-action drop
commit; save
```

### Output Chain

This manages traffic originating from the VyOS router. If the router itself makes outbound requests (such as NTP synchronization or software updates), they are processed through the OUTPUT chain.

```bash
set firewall ipv4 output filter default-action accept 
commit; save
```

### Forward Chain

This handles traffic passing through the router but not directed to or from it. If VyOS is acting as a router between networks, the FORWARD chain determines which packets are allowed to pass between them.

```bash hl_lines="8"
set firewall ipv4 forward filter rule 5 action 'drop'
set firewall ipv4 forward filter rule 5 state 'invalid'
set firewall ipv4 forward filter rule 5 description 'Drop invalid state packets'
set firewall ipv4 forward filter rule 20 action 'accept'
set firewall ipv4 forward filter rule 20 description 'Allow Return traffic through the router'
set firewall ipv4 forward filter rule 20 state 'established'
set firewall ipv4 forward filter rule 20 state 'related'
set firewall ipv4 forward filter rule 20 inbound-interface name [WAN_INTERFACE]
set firewall ipv4 forward filter rule 1000 action 'accept'
set firewall ipv4 forward filter rule 1000 description 'Allow all traffic from LAN interface'
set firewall ipv4 forward filter rule 1000 inbound-interface name br0
set firewall ipv4 forward filter default-action drop
commit; save
```

??? tip "Allow ICMP/Ping on WAN"
    By default, the `drop` default-action blocks all ICMP on the WAN interface, including useful diagnostics like ping and Path MTU Discovery. If you want to allow limited ICMP traffic on the WAN, add the following rules to the input chain:

    ```bash hl_lines="5 10"
    set firewall ipv4 input filter rule 15 action 'accept'
    set firewall ipv4 input filter rule 15 description 'Allow ICMP echo requests (ping) on WAN'
    set firewall ipv4 input filter rule 15 protocol 'icmp'
    set firewall ipv4 input filter rule 15 icmp type-name 'echo-request'
    set firewall ipv4 input filter rule 15 inbound-interface name [WAN_INTERFACE]
    set firewall ipv4 input filter rule 16 action 'accept'
    set firewall ipv4 input filter rule 16 description 'Allow ICMP destination-unreachable on WAN'
    set firewall ipv4 input filter rule 16 protocol 'icmp'
    set firewall ipv4 input filter rule 16 icmp type-name 'destination-unreachable'
    set firewall ipv4 input filter rule 16 inbound-interface name [WAN_INTERFACE]
    commit; save
    ```

## System

### User

For security best practices, remove the default `vyos` user and create a new one with administrative privileges. The password will be encrypted when committing the changes.

```bash hl_lines="1"
set system login user [USERNAME] authentication plaintext-password [PASSWORD]
commit; save
```

Login with your new user account to verify it works. Then delete the `vyos` user account.

```bash
delete system login user vyos
commit; save
```

### SSH

Enable SSH access and restrict it to the LAN interface so the router cannot be accessed remotely from the WAN.

```bash
set service ssh port 22 # (1)!
set service ssh listen-address 192.168.1.1 # (2)!
set service ssh disable-host-validation # (3)!
commit; save
```

1. :material-network: **Port** - Enables the SSH service on the default port
2. :material-server: **Listen Address** - Binds SSH to the LAN address only, preventing WAN access
3. :material-dns: **Host Validation** - Disable IP Address to Hostname lookup

To add your public key for SSH key-based authentication:

```bash hl_lines="2 3"
set service ssh disable-password-authentication # (1)!
set system login user [USERNAME] authentication public-keys [KEY_NAME] key [PUBLIC_KEY]
set system login user [USERNAME] authentication public-keys [KEY_NAME] type ssh-rsa
commit; save
```

1. :material-lock: **Password Auth** - Disables password authentication, requiring SSH key-based login


### Hostname

Set the hostname of the system to something easily identifiable.

```bash hl_lines="1"
set system host-name [HOSTNAME]
commit; save
```

### NTP

By default, VyOS acts as an NTP server for clients. This is usually unnecessary for home use, so disable it.

```bash
delete service ntp allow-client
commit; save
```

VyOS defaults to NTP servers in the US, Germany, and Singapore (AWS). For better accuracy, use servers closer to your location.

```bash hl_lines="8"
delete service ntp server time1.vyos.net
delete service ntp server time2.vyos.net
delete service ntp server time3.vyos.net
set service ntp server 0.pool.ntp.org
set service ntp server 1.pool.ntp.org
set service ntp server 2.pool.ntp.org
set service ntp server 3.pool.ntp.org
set system time-zone [TIMEZONE]
commit; save
```

Now your VyOS router is fully configured and ready to power your homelab! With a secure and efficient network in place, you can focus on building and exploring your homelab projects. Happy networking!

[vyos]: https://vyos.io
[downloading]: https://vyos.net/get/stream/
[ssh]: #ssh
