# Edge Security and Site-to-Site VPN for Aanchal Boutique
**Task 1: Virtual Network Foundation and Addressing**

## Short Note: What Was Done and Why
For **Aanchal Boutique**, I designed the two-site virtual network foundation (**Site A** and **Site B**) using non-overlapping IPv4 subnets (`192.168.10.0/24` and `192.168.20.0/24`), planned two VirtualBox internal networks (`SiteA_LAN` and `SiteB_LAN`), assigned static IPs to the **OPNsense** WAN/LAN interfaces and **Ubuntu Server** clients, and set each OPNsense LAN IP as the client's default gateway. Non-overlapping subnets were chosen so both locations remain isolated now and avoid IP conflicts when the site-to-site IPsec VPN is configured in later tasks. Due to hardware limits (mobile access and a 4 GB RAM laptop), live multi-VM execution and end-to-end ping verification were not completed locally and remain to be run on a higher-RAM host.

---

## 1. Topology Diagram

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

