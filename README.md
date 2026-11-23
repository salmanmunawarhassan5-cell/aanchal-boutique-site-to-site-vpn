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

