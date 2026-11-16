# pfsense-openvpn-traffic-analysis
Configuring a pfSense OpenVPN server and analyzing encrypted tunnel traffic and decapsulated ICMP packets using Wireshark.
# pfSense OpenVPN VPN and Traffic Analysis

## Overview

This project involved configuring a pfSense firewall as an OpenVPN server and connecting a Kali Linux machine as a VPN client. The goal was to establish a certificate-authenticated VPN tunnel, route client internet traffic through the tunnel, and analyze how traffic appears before and after VPN encapsulation.

Wireshark was used to capture traffic at two points:
- Outer capture: OpenVPN traffic exchanged between the client and server.
- Inner capture: Decapsulated ICMP traffic passing through the VPN tunnel.

Comparing these captures provided a way to verify VPN routing and observe the difference between encapsulated VPN traffic and the original network packets.

## Lab Topology

`
                  Internet
                     |
              VMware NAT (VMnet8)
                192.168.99.0/24
                     |
              pfSense WAN (em0)
                192.168.99.130
                     |
                pfSense CE
              OpenVPN Server
                     |
              pfSense LAN (em1)
                192.168.250.1
                     |
              VMware LAN Segment
                    lan1
                     |
                Kali Linux
              192.168.250.128
                     |
               OpenVPN Client
                10.8.0.2
                     |
                VPN Tunnel
                 10.8.0.0/24
``

## Network Configuration

| Component | Interface | IP Address |
|---|---|---|
| pfSense WAN | em0 | 192.168.99.130/24 |
| pfSense LAN | em1 | 192.168.250.1/24 |
| Kali Linux | eth0 | 192.168.250.128/24 |
| VPN network | tun0 | 10.8.0.0/24 |
| Kali VPN address | tun0 | 10.8.0.2 |

## Implementation

### pfSense Firewall

Configured pfSense Community Edition as the firewall and VPN server.

Key configuration:
- WAN interface using DHCP.
- LAN interface using static addressing.
- OpenVPN server using UDP on port 1194.
- VPN tunnel network: `10.8.0.0/24`.
- Local network: `192.168.250.0/24`.
- Certificate-based client authentication.
- TLS authentication enabled.
- IPv4 gateway redirection enabled to route client internet traffic through the VPN.
- Firewall rules configured to permit the required LAN and VPN traffic.

### VPN Authentication

Created a certificate authority, server certificate, and client certificate for authentication.

The OpenVPN Client Export package was used to generate the Kali client configuration. The exported client configuration was imported into Kali to establish the VPN connection.

### Kali Linux Client

The VPN client successfully connected to the pfSense server and received the tunnel address `10.8.0.2`.

The routing table was checked to confirm that internet-bound IPv4 traffic was directed through `tun0`.

## VPN Validation

Several tests were performed to verify the tunnel:

**1. VPN connection**

OpenVPN reported:

``
Peer Connection Initiated
TUN/TAP device tun0 opened
Initialization Sequence Completed
```

2. Routing verification


```
ip route get 8.8.8.8
```

Result:

``
8.8.8.8 via 10.8.0.1 dev tun0 src 10.8.0.2
```

This confirmed that traffic destined for the internet was routed through the VPN interface.

**3. Connectivity test

Command:

```
ping -c 5 8.8.8.8
```

Result:
- 5 packets transmitted
- 5 packets received
- 0% packet loss

## Traffic Capture and Analysis

Two separate packet captures were collected using `tcpdump` and examined in Wireshark.

### 1. Outer VPN Traffic

<text>File: `captures/vpn-capture.pcap`</text>

Capture command:

```
sudo tcpdump -i any -nn -w ~/Desktop/vpn-capture.pcap udp port 1194
```

The capture contained 22 OpenVPN packets exchanged between:

- Client: `192.168.250.128`
- Server: `192.168.250.1`
- Protocol: UDP
- Port: 1194

Wireshark identified the packets as `P_DATA_V2`, indicating OpenVPN data-channel traffic.

The original ICMP packet contents were not directly visible in this outer capture because they were encapsulated within the VPN data channel.

![Outer OpenVPN traffic](screenshots/11-outer-traffic-wireshark.png)

### 2. Inner VPN Traffic

<text>File: `captures/vpn-inner.pcap`</text>

Capture command:

```
sudo tcpdump -i tun0 -nn -w ~/Desktop/vpn-inner.pcap icmp
```

The capture showed the original ICMP traffic after it had been decapsulated by the VPN client.

The 10 captured frames consisted of:
- 5 ICMP echo requests from `10.8.0.2` to `8.8.8.8`.
- 5 ICMP echo replies from `8.8.8.8` to `10.8.0.2`.

All five requests received corresponding replies.

![Inner ICMP traffic](screenshots/12-inner-traffic-wireshark.png)

## Key Findings

Comparing the two captures showed the difference between the traffic carried by the VPN tunnel and the original packets inside it.

| Observation | Outer Capture | Inner Capture |
|---|---|---|
| Capture point | VPN transport traffic | VPN tunnel interface |
| Protocol | OpenVPN over UDP | ICMP |
| Source | 192.168.250.128 | 10.8.0.2 |
| Destination | 192.168.250.1 | 8.8.8.8 |
| Packet visibility | OpenVPN data-channel packets | Original ICMP requests and replies |
| Purpose | Observe VPN traffic | Verify tunneled traffic |

The outer capture showed OpenVPN data-channel packets exchanged between the client and server, while the inner capture exposed the original ICMP packets after decapsulation.

Together, the captures provided evidence that the VPN tunnel was carrying client traffic and that the client could successfully communicate with an external destination through the tunnel.

## Repository Contents

| Path | Description |
|---|---|
| `report/` | Formal lab report with configuration details and packet analysis |
| `captures/` | Outer and inner PCAP files |
| `screenshots/` | Configuration, connection, routing, and Wireshark evidence |

## Skills Demonstrated

- pfSense firewall configuration
- OpenVPN server and client setup
- Certificate-based authentication
- Firewall rules and network segmentation
- VPN routing and traffic redirection
- Wireshark packet analysis
- tcpdump traffic capture
- Network troubleshooting and connectivity validation

## Disclaimer

This project was conducted in an isolated virtual lab using VMware, pfSense Community Edition, and Kali Linux. It was created for educational purposes. No production networks or systems were involved.
