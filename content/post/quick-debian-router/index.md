---
title: Setting Up Debian as a Gateway Router
description: Turn Debian into a basic router for traffic analysis, NAT, etc.
date: 2024-11-17 17:24:00+0000
draft: false
toc: false
---

Turning Debian into a network router involves configuring the system to forward IP packets between network interfaces and applying appropriate IP routing rules. This step-by-step guide walks you through the setup on a Debian 12 VM.

### Summary

1. **Start with foundational functionality (IP forwarding)**
2. **Set up essential connectivity (network interfaces)**
3. **Optional: Set up NAT**
4. **Optional: Configure routing rules**
5. **Optional: Install a DHCP server**

### **Prerequisites**

Make sure your VM has at least two network interfaces:

- One for the external network (e.g., connected to the internet or another external network).
- One for the internal network (e.g., the LAN to be routed).

You can check your network interfaces with:

```sh
ip a
```

### 1. **Enable IP Forwarding**

1. Edit the sysctl configuration to enable IP forwarding permanently:

```sh
sudo sed -i '$ a net.ipv4.ip_forward=1' /etc/sysctl.conf
```

Alternatively, open the sysctl configuration file `/etc/sysctl.conf` and locate or add the line `net.ipv4.ip_forward = 1`

2. Apply the changes immediately:

```sh
sudo sysctl -p
```

To verify that IP forwarding is active:

```sh
cat /proc/sys/net/ipv4/ip_forward
```

It should return `1`.

### 2. Configuring Network Interfaces with `systemd-networkd`

1. **Verify that `systemd-networkd` is active**

Ensure `systemd-networkd` is running and enabled:

```sh
sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd
```

If you're switching from `ifupdown` or another network manager, disable and remove it first:

```sh
sudo systemctl disable networking
sudo apt remove ifupdown
```

2. **Set Up the Network Configuration**

Define `.network` and `.netdev` files in `/etc/systemd/network/`. Each file should correspond to a network interface.

Example:

- External interface (`eth0`): Create `/etc/systemd/network/10-external.network`:

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
```

- Internal interface (`eth1`): Create `/etc/systemd/network/20-internal.network`:

```ini
[Match]
Name=eth1

[Network]
Address=192.168.1.1/24
```

3. **Restart `systemd-networkd`** Apply the configuration:

```sh
sudo systemctl restart systemd-networkd
```

4. **Verify Configuration** Use `networkctl` to check the status of the interfaces:

```sh
networkctl status
```

You should see both interfaces correctly configured.

### 3. **Optional: Set Up NAT**

To allow machines on the internal network to access external networks through the router, configure Network Address Translation (NAT) using  `nftables`.

1. Make sure `nftables` is running and enabled:

```sh
sudo systemctl enable nftables
sudo systemctl start nftables
```

2. Create a basic configuration file (e.g., `/etc/nftables.conf`):

```nft
table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        oifname "eth0" masquerade
    }
}
```

3. Apply the rules:

```sh
sudo nft -f /etc/nftables.conf
```

### 4. **Optional: Configure Routing Rules**

Add any required static routes using `ip route` commands:

```sh
sudo ip route add [destination_network] via [gateway_ip] dev [interface]
```

Example:

```sh
sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth1
```

### 5. **Optional: Install a DHCP Server**

To dynamically assign IPs to devices on the internal network, install and configure a DHCP server. In case of turning a Debian VM into a basic router, `dnsmasq` is a practical, efficient choice.

1. **Install dnsmasq**

```sh
sudo apt update
sudo apt install dnsmasq
```

2. **Configure `dnsmasq`** Edit the configuration file:

```sh
sudo nano /etc/dnsmasq.conf
```

Add or modify the following lines for your setup:

- Specify the internal interface to listen on:

```plaintext
interface=eth1
```

- Define the DHCP range for the internal network:

```plaintext
dhcp-range=192.168.1.100,192.168.1.200,12h
```

  This assigns IP addresses from `192.168.1.100` to `192.168.1.200` with a lease time of 12 hours.

- Set the gateway (router) and DNS server (can be the router itself or another server):

```sh
dhcp-option=option:router,192.168.1.1
dhcp-option=option:dns-server,8.8.8.8
```

3. **Restart dnsmasq** Apply the configuration:

```sh
sudo systemctl restart dnsmasq
```

4. **Enable dnsmasq at Boot** Ensure `dnsmasq` starts automatically:

```sh
sudo systemctl enable dnsmasq
```

5. **Verify dnsmasq is Running** Check the status of the service:

```sh
sudo systemctl status dnsmasq
```

Ensure it’s listening on the correct interface:

```sh
sudo netstat -ulnp | grep dnsmasq
```

With these steps, Debian is now a fully functional network router, capable of managing traffic between your internal and external networks.