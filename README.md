# Edge Security and Site-to-Site VPN for Aanchal Boutique


# Task 1: Virtual Network Foundation and Addressing — Aanchal Boutique

## What Was Done and Why

For **Aanchal Boutique**, a two-site virtual network foundation (**Site A — Main Office** and **Site B — Branch Office**) was designed using non-overlapping IPv4 subnets (`192.168.10.0/24` and `192.168.20.0/24`), two VirtualBox internal networks (`SiteA_LAN` and `SiteB_LAN`), static IP addressing across OPNsense WAN/LAN interfaces and Ubuntu Server clients, and each site's OPNsense LAN IP configured as its local client's default gateway.

Non-overlapping subnets were selected to keep Site A and Site B logically isolated during initial setup while preventing IP address conflicts when configuring the site-to-site IPsec VPN in later tasks.

---

## Topology Diagram

```text
Site A (Main Office)          Site B (Branch Office)
Subnet: 192.168.10.0/24       Subnet: 192.168.20.0/24
Net: SiteA_LAN                Net: SiteB_LAN

[Ubuntu-Client-SiteA]         [Ubuntu-Client-SiteB]
IP: 192.168.10.10/24          IP: 192.168.20.10/24
GW: 192.168.10.1              GW: 192.168.20.1
         |                             |
   (SiteA_LAN)                   (SiteB_LAN)
         |                             |
[OPNsense-FW-SiteA]           [OPNsense-FW-SiteB]
LAN: 192.168.10.1/24          LAN: 192.168.20.1/24
WAN: 10.0.2.15/24             WAN: 10.0.2.16/24
         \                             /
          +------ [ WAN / NAT ] ------+

ping -c 4 192.168.10.1
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.42 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=0.36 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=64 time=0.39 ms
64 bytes from 192.168.10.1: icmp_seq=4 ttl=64 time=0.35 ms

--- 192.168.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3048ms
rtt min/avg/max/mdev = 0.350/0.380/0.420/0.028 ms

ping -c 4 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.44 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=0.37 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=0.40 ms
64 bytes from 192.168.20.1: icmp_seq=4 ttl=64 time=0.38 ms

--- 192.168.20.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3051ms
rtt min/avg/max/mdev = 0.370/0.397/0.440/0.027 ms
