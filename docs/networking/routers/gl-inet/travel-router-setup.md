---
title: Travel Router Setup
categories:
- Networking
- Router
date: 2025-04-05
description: Learn how to set up your travel router for secure and reliable internet
  on the go. Stay connected anywhere!
draft: false
tags:
- travel router
---


Whether you travel frequently or just want a secure, stable connection on the go, a travel router is a game-changer. This guide covers everything from choosing the right model to securing and optimizing your connection, helping you stay connected wherever you are.

??? info "Prerequisites"
    - GL.iNet Beryl AX router or similar GL.iNet device running OpenWRT
    - Access to hotel/public Wi-Fi or Ethernet connection
    - Basic understanding of networking concepts

## Hardware 

When it comes to a travel router, you want it to light and small. You don't want to carry your big heavy router from home with you. 

In terms of the hardware GL-Inet makes small Wifi routers in the size of a pack of cards. 
One of the best bang for your buck devices is the Beryl AX. The GL.iNet Beryl AX (GL-MT3000) is a compact Wi-Fi 6 travel router equipped with the following specifications:​

| Component | Specification |
| --------- | ------------- |
| **Processor** | MediaTek MT7981B |
| **Memory** | 512MB DDR4 RAM |
| **Storage** | 256MB NAND Flash |
| **Wi-Fi Standards** | 802.11a/b/g/n/ac/ax (Wi-Fi 6) |
| **Wi-Fi Speeds** | 2.4 GHz: 574 Mbps<br>5 GHz: 2402 Mbps |
| **Antennas** | 2× retractable external antennas |
| **Ethernet Ports** | 1× 2.5 Gbps WAN<br>1× 1 Gbps LAN |
| **USB** | 1× USB 3.0 Type-A |
| **Power** | 5V/3A via USB Type-C |
| **Dimensions** | 105 × 68 × 24 mm |
| **Weight** | ~130g |

Other reason to pick GL.iNet Beryl AX is that it runs OpenWRT, this applies to all GL.iNet devices. Their routers ship with a user-friendly UI, but you still have full access to the standard OpenWrt LuCI interface if you want deeper customization. This makes them great for both beginners and advanced users who need flexibility for things like VPNs, VLANs, ad-blocking, and more. 

## Initial setup

!!! info
    This setup is done with version `4.7.0` installed

During the initial setup you can either connect via a ethernet cable or to the default Wireless network of the Beryl. 

In order to login to the Beryl you have to go  `https://192.168.8.1`. This will prompt you a setup wizard to configure a login password to login to the UI.
As well as a the name and password for the wireless network. For now I leave the Wireless settings default, as I will change more settings later on. 

## Connect to Internet 

### Ethernet 

For the best possible experience, it's best to connect the Beryl with an Ethernet cable when you have the possibility. Some hotels and Airbnbs have an Ethernet cable connected to the TV for streaming services. If you don't need those, for example, when you bring your own streaming device, you can connect this cable to the WAN port of the Beryl. The Beryl will automatically pick up an IP address from the DHCP server.  

Please note that there might be security measures in place against this. If the Beryl does not get an IP address, the best option might be to connect the Beryl via WiFi as explained below.

### Connect to WiFi 

On the main dashboard, locate the **Repeater** section and click **Connect** to join the Beryl to an existing wireless network. Select the desired network, enter the password if required, and keep all other settings as default.  

If an authentication portal is needed, open a browser on your device to complete the login process.

## Wireless network

By default, the Beryl has separate networks for 2.4GHz and 5GHz. To simplify things, we'll give them the same name, allowing devices to automatically choose the best band based on signal strength and coverage.  

**Steps to configure**:

1. **Go to** the **Wireless** section in the side menu.
2. **Click on** `Modify`.
3. **Set the TX Power to Medium**
4. **Change the SSID and Password** to your liking.
5. **Set Wi-Fi Security** to **`WPA2-PSK/WPA3-SAE`** for optimal security.
6. **Click Apply.**

**Configuration explained:**

- **Same SSID for both bands**: Using identical network names for 2.4GHz and 5GHz allows devices to automatically select the best band based on signal strength and capabilities
- **TX Power (Medium)**: Reduces power consumption and signal interference in small spaces like hotel rooms while maintaining adequate coverage
- **WPA2-PSK/WPA3-SAE**: Provides the highest security standard with backward compatibility for older devices that don't support WPA3

Repeat these steps for the 2.4GHz network.

Once you've updated the 2.4GHz network settings, your Beryl is ready to go—use it anywhere you like!
