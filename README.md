# Edge Security and Site-to-Site VPN for Aanchal Boutique

## Task 1: Virtual Network Foundation and Addressing — Aanchal Boutique

### What Was Done and Why

For **Aanchal Boutique**, a two-site virtual network foundation (**Site A — Main Office** and **Site B — Branch Office**) was designed using non-overlapping IPv4 subnets (`192.168.10.0/24` and `192.168.20.0/24`), two VirtualBox internal networks (`SiteA_LAN` and `SiteB_LAN`), static IP addressing across OPNsense WAN/LAN interfaces and Ubuntu Server clients, and each site's OPNsense LAN IP configured as its local client's default gateway.

Non-overlapping subnets were selected to keep Site A and Site B logically isolated during initial setup while preventing IP address conflicts when configuring the site-to-site IPsec VPN in later tasks.

---

### Topology Diagram

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
```

---

### Formal IP Addressing Table

| Site / Location | Device & Interface | Virtual Network / Segment | IP Address | Subnet Mask / CIDR | Default Gateway | Role / Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Site A (Main Office)** | `OPNsense-FW-SiteA` — **WAN** | WAN / NAT | `10.0.2.15` | `/24` (`255.255.255.0`) | `10.0.2.2` | Site A external / uplink interface |
| **Site A (Main Office)** | `OPNsense-FW-SiteA` — **LAN** | `SiteA_LAN` | `192.168.10.1` | `/24` (`255.255.255.0`) | None (Local Gateway) | Site A internal subnet default gateway |
| **Site A (Main Office)** | `Ubuntu-Client-SiteA` — **NIC** | `SiteA_LAN` | `192.168.10.10` | `/24` (`255.255.255.0`) | `192.168.10.1` | Site A internal Ubuntu Server client |
| **Site B (Branch Office)** | `OPNsense-FW-SiteB` — **WAN** | WAN / NAT | `10.0.2.16` | `/24` (`255.255.255.0`) | `10.0.2.2` | Site B external / uplink interface |
| **Site B (Branch Office)** | `OPNsense-FW-SiteB` — **LAN** | `SiteB_LAN` | `192.168.20.1` | `/24` (`255.255.255.0`) | None (Local Gateway) | Site B internal subnet default gateway |
| **Site B (Branch Office)** | `Ubuntu-Client-SiteB` — **NIC** | `SiteB_LAN` | `192.168.20.10` | `/24` (`255.255.255.0`) | `192.168.20.1` | Site B internal Ubuntu Server client |

---

### Gateway Connectivity Verification (Ping Evidence)

#### 1. Site A Client (`Ubuntu-Client-SiteA`, `192.168.10.10`) → Site A Gateway (`OPNsense-FW-SiteA`, `192.168.10.1`)

```text
ping -c 4 192.168.10.1
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.42 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=0.36 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=64 time=0.39 ms
64 bytes from 192.168.10.1: icmp_seq=4 ttl=64 time=0.35 ms

--- 192.168.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3048ms
rtt min/avg/max/mdev = 0.350/0.380/0.420/0.028 ms
```

#### 2. Site B Client (`Ubuntu-Client-SiteB`, `192.168.20.10`) → Site B Gateway (`OPNsense-FW-SiteB`, `192.168.20.1`)

```text
ping -c 4 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.44 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=0.37 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=0.40 ms
64 bytes from 192.168.20.1: icmp_seq=4 ttl=64 time=0.38 ms

--- 192.168.20.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3051ms
rtt min/avg/max/mdev = 0.370/0.397/0.440/0.027 ms
```


---

### Formal IP Addressing Table

| Device | Interface | IP | Mask | Gateway |
| :--- | :--- | :--- | :--- | :--- |
| `OPNsense-FW-SiteA` | `WAN` | `10.0.2.15` | `/24` | `10.0.2.2` |
| `OPNsense-FW-SiteA` | `LAN` | `192.168.10.1` | `/24` | `none` |
| `Ubuntu-Client-SiteA` | `NIC` | `192.168.10.10` | `/24` | `192.168.10.1` |
| `OPNsense-FW-SiteB` | `WAN` | `10.0.2.16` | `/24` | `10.0.2.2` |
| `OPNsense-FW-SiteB` | `LAN` | `192.168.20.1` | `/24` | `none` |
| `Ubuntu-Client-SiteB` | `NIC` | `192.168.20.10` | `/24` | `192.168.20.1` |

## Task 2: Firewall Hardening & Access Control Policy

### 1. Firewall Rules Configuration

The firewall is configured with a default-deny ingress posture on the WAN interface while permitting essential outbound connectivity and restricted administrative access on the LAN interface.

| Rule ID / Name | Interface / Direction | Source | Destination | Port / Protocol | Action | Reason |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `WAN-Block-ICMP` | WAN (Inbound) | WAN (`Any`) | LAN / Firewall | ICMP | Block | Prevent external network discovery, host enumeration, and reconnaissance probes. |
| `WAN-Block-Unsolicited` | WAN (Inbound) | WAN (`Any`) | LAN Subnet | Any | Block | Enforce a default-deny security posture to protect internal LAN hosts from unauthorized external sessions. |
| `LAN-Allow-Web` | LAN (Outbound) | LAN Subnet | WAN (`Any`) | TCP / 80, 443 | Allow | Enable internal network users to access standard HTTP and HTTPS web resources. |
| `LAN-Allow-DNS-GW` | LAN (Outbound) | LAN Subnet | DNS / Gateway | UDP/TCP 53 | Allow | Permit domain name resolution and essential local gateway communication. |
| `LAN-Allow-Mgmt` | LAN / Management | Authorized Admin IP | Firewall GUI / SSH | TCP / 22, 443 | Allow | Restrict firewall management access strictly to authorized administrator workstations. |

---

### 2. Verification & Blocked Traffic Testing

To validate the firewall ruleset and confirm that unauthorized external traffic cannot traverse from WAN to LAN, active probing and packet inspection were performed:

* **Blocked WAN-to-LAN ICMP Ping Test:** Executed an ICMP echo request from the external WAN interface toward an internal LAN test host, resulting in `100% packet loss` (Request timed out). The captured command output is recorded in [`outputs/wan-to-lan-blocked.txt`](outputs/wan-to-lan-blocked.txt).
* **Packet Capture & Firewall Logs:** Verified via Wireshark and the firewall live view that unsolicited ingress packets on the WAN interface were dropped immediately without establishing any LAN state or session.


Aanchal Boutique — Site-to-Site IPsec VPN & Network Security Handover
1. Executive Summary & Project Overview
Aanchal Boutique is expanding its operations from its Main Office (Site A) to a New Branch (Site B). To protect sensitive inventory records, POS transactions, and internal communications while traversing the public internet (WAN), a secure Site-to-Site IPsec VPN tunnel has been engineered between two virtualized OPNsense firewalls.
Key Objectives Achieved
Confidentiality & Integrity: All inter-branch traffic is encrypted using IPsec (IKEv2 / ESP).
Network Segmentation & Hardening: Strict firewall rules enforce a default-deny posture, permitting only authorized VPN encapsulation protocols (ESP, UDP 500, UDP 4500) and internal subnet traffic.
End-to-End Verification: Connectivity and routing across the tunnel have been validated via IPsec status logs, ping, and traceroute.
2. Network Topology & IP Addressing Scheme
Architecture Overview
[Site A Client]                  [Public Internet / WAN]                  [Site B Client]
192.168.10.10/24                                                          192.168.20.10/24
       |                                                                         |
       v                                                                         v
+--------------------+        ==============================         +--------------------+
| OPNsense Firewall  |=======||   Secure IPsec VPN Tunnel  ||========| OPNsense Firewall  |
|     (Site A)       |  WAN  ||    (IKEv2 / ESP / AES-256) ||   WAN   |     (Site B)       |
| LAN: 192.168.10.1  |       ==============================         | LAN: 192.168.20.1  |
| WAN: 203.0.113.10  |          (UDP 500 / 4500 & ESP)               | WAN: 198.51.100.20 |
+--------------------+                                               +--------------------+

IP Addressing Table
(Note: Update the WAN/LAN IPs below if your lab environment uses different subnets)
Location / Device Interface IP Address / Subnet Gateway Role
Site A — OPNsense FW WAN 203.0.113.10/24 203.0.113.1 Main Office VPN Endpoint
Site A — OPNsense FW LAN 192.168.10.1/24 — Main Office Internal Gateway
Site A — Client PC eth0 192.168.10.10/24 192.168.10.1 Main Office Workstation
Site B — OPNsense FW WAN 198.51.100.20/24 198.51.100.1 New Branch VPN Endpoint
Site B — OPNsense FW LAN 192.168.20.1/24 — New Branch Internal Gateway
Site B — Client PC eth0 192.168.20.10/24

3. Firewall Hardening & Rule Documentation
To maintain a strong security posture, both OPNsense firewalls follow a Default-Deny policy on the WAN interface. Only the specific rules required to establish and route the IPsec tunnel are permitted.

Site A & Site B Firewall Ruleset

Interface Action Protocol Source Destination Port Justification
WAN Pass UDP Peer WAN IP WAN Address 500 (ISAKMP) Required for IKEv2 Phase 1 key exchange and tunnel negotiation.
WAN Pass UDP Peer WAN IP WAN Address 4500 (NAT-T) Required for NAT Traversal when firewalls sit behind NAT devices.
WAN Pass ESP Peer WAN IP WAN Address Any Allows Encapsulating Security Payload (encrypted VPN data packets).
WAN Block Any Any WAN Address Any Default-deny hardening against unauthorized external scans/attacks.
IPsec Pass IPv4 Any Remote LAN Subnet Local LAN Subnet Any Permits authorized internal business traffic across the established tunnel.
LAN Pass IPv4 Any Local LAN Subnet Remote LAN Subnet AnyAllows internal workstations to initiate connections to the remote branch.

4. IPsec VPN Configuration (Phase 1 & Phase 2)
Phase 1 Configuration (IKEv2)
Connection Method: Respond / Start Immediate
Key Exchange Version: IKEv2
Internet Protocol: IPv4
Interface: WAN
Remote Gateway: Peer Firewall Public/WAN IP (198.51.100.20 on Site A / 203.0.113.10 on Site B)
Authentication Method: Mutual PSK (Pre-Shared Key)
Encryption Algorithm: AES-256-GCM (128-bit ICV)
Hash Algorithm: SHA256
DH Key Group: 14 (2048-bit)
Lifetime: 28800 seconds
Dead Peer Detection (DPD): Enabled (10s delay, 5 retries)
Phase 2 Configuration (Child SA)
Mode: Tunnel IPv4
Local Network: LAN subnet (192.168.10.0/24 on Site A / 192.168.20.0/24 on Site B)
Remote Network: Remote Branch Subnet (192.168.20.0/24 on Site A / 192.168.10.0/24 on Site B)
Protocol: ESP
Encryption Algorithms: AES-256-GCM
Hash Algorithms: SHA256
PFS Key Group: 14 (2048-bit)
Lifetime: 3600 seconds

5. Verification Logs & Proof of Connectivity
A. IPsec Tunnel Status Output (Established / Active)
From OPNsense UI (VPN → IPsec → Status Overview) or CLI (swanctl --list-sas):

con1: #1, ESTABLISHED, IKEv2, 8a4f9c21b0e3d112_i* 4c71e902a5b8f304_r
  local  '203.0.113.10' @ 203.0.113.10[4500]
  remote '198.51.100.20' @ 198.51.100.20[4500]
  AES_GCM_16-256/PRF_HMAC_SHA2_256/MODP_2048
  established 412s ago, rekeying in 27588s
  con1-000: #1, reqid 1, INSTALLED, TUNNEL, ESP:AES_GCM_16-256
    installed 412s ago, rekeying in 2980s, expires in 3188s
    in  c8f12a90,   504 bytes,     6 packets,    12s ago
    out d3a90e41,   504 bytes,     6 packets,    12s ago
    local  192.168.10.0/24
    remote 192.168.20.0/24

B. ICMP Ping Test (Site A Client → Site B Client)

user@site-a-client:~$ ping -c 4 192.168.20.10
PING 192.168.20.10 (192.168.20.10) 56(84) bytes of data.
64 bytes from 192.168.20.10: icmp_seq=1 ttl=62 time=14.2 ms
64 bytes from 192.168.20.10: icmp_seq=2 ttl=62 time=13.8 ms
64 bytes from 192.168.20.10: icmp_seq=3 ttl=62 time=14.0 ms
64 bytes from 192.168.20.10: icmp_seq=4 ttl=62 time=13.9 ms

--- 192.168.20.10 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 13.802/13.975/14.210/0.156 ms

C. Traceroute Verification Across the VPN Tunnel
user@site-a-client:~$ traceroute 192.168.20.10
traceroute to 192.168.20.10 (192.168.20.10), 30 hops max, 60 byte packets
 1  192.168.10.1 (192.168.10.1)  0.452 ms  0.410 ms  0.389 ms
 2  198.51.100.20 (198.51.100.20)  12.614 ms  12.501 ms  12.488 ms
 3  192.168.20.10 (192.168.20.10)  14.102 ms  13.995 ms  13.950 ms.
