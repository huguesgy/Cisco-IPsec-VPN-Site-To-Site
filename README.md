# 🛡️ Project : Site-to-Site IPsec VPN & NAT Bypass (Cisco IOS)

[![Language: French](https://img.shields.io/badge/Language-Français-blue.svg)](README_FR.md)
![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![GNS3](https://img.shields.io/badge/Simulator-GNS3-orange?style=for-the-badge)
![VPN](https://img.shields.io/badge/Security-IPsec%20VPN-green?style=for-the-badge)

> 🇫🇷 *Pour consulter la documentation en Français, [cliquez ici](README_FR.md).*

---

## 📌 Project Overview
This project focuses on designing, deploying, and securing a WAN interconnection between a **Headquarters (R1)** and a remote **Branch Office (R2)** across an untrusted simulated Internet service provider (**ISP**).

Key Features:
1. **Inter-site Data Confidentiality** via an encrypted **Site-to-Site IPsec VPN** tunnel (ISAKMP / IKEv1 + IPsec ESP-AES).
2. **Local Internet Access** for both sites using Dynamic Network Address Translation (**NAT Overload / PAT**).
3. **NAT Bypass / Exemption** to ensure private corporate traffic destined for the VPN tunnel is not altered by NAT translation rules.

---

## 📐 Network Topology

```text
 [ HQ LAN ]                                                                                        [ Branch LAN ]
 PC1 (192.168.10.1)                                                                              PC2 (192.168.20.1)
         |                                                                                               |
         v                                                                                               v
   +-----------+        WAN 1 (203.0.113.0/30)        +-------+       WAN 2 (198.51.100.0/30)       +---------------+
   |  R1 (HQ)  |-------------------------------------|  ISP  |-------------------------------------|  R2 (Branch)  |
   +-----------+                                     +-------+                                     +---------------+
                                                         |
                                                         v
                                              [ Simulated Internet ]
                                                   DNS (8.8.8.8)
```

---

## 📊 IP Addressing Plan

| Device | Interface | IP Address / Mask | Role / Description |
| :--- | :--- | :--- | :--- |
| **R1 (HQ)** | `Gi0/0` | `192.168.10.254/24` | HQ LAN Gateway |
| **R1 (HQ)** | `Gi0/1` | `203.0.113.1/30` | WAN Interface to ISP |
| **R2 (Branch)** | `Gi0/0` | `192.168.20.254/24` | Branch LAN Gateway |
| **R2 (Branch)** | `Gi0/1` | `198.51.100.1/30` | WAN Interface to ISP |
| **ISP** | `Gi0/0` | `203.0.113.2/30` | WAN Interface to R1 |
| **ISP** | `Gi0/1` | `198.51.100.2/30` | WAN Interface to R2 |
| **ISP** | `Loopback0`| `8.8.8.8/32` | Simulated Internet DNS Server |

---

## 🛠️ Network Device Configurations (Cisco IOS)

### 1. Router R1 (HQ) Configuration

```cisco
hostname R1

! --- Interface Configuration ---
interface GigabitEthernet0/0
 ip address 192.168.10.254 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.252
 ip nat outside
 crypto map MY_MAP
 no shutdown

! --- Default WAN Route ---
ip route 0.0.0.0 0.0.0.0 203.0.113.2

! --- 1. NAT Bypass / Exemption ---
access-list 110 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip 192.168.10.0 0.0.0.255 any
ip nat inside source list 110 interface GigabitEthernet0/1 overload

! --- 2. IPsec Phase 1 (ISAKMP) ---
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key SecretKey123 address 198.51.100.1

! --- 3. IPsec Phase 2 (Transform-Set & Interesting Traffic ACL) ---
crypto ipsec transform-set MY_SET esp-aes esp-sha-hmac
access-list 120 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255

! --- 4. Crypto Map Assembly & Binding ---
crypto map MY_MAP 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set MY_SET
 match address 120
```

---

### 2. Router R2 (Branch) Configuration

```cisco
hostname R2

! --- Interface Configuration ---
interface GigabitEthernet0/0
 ip address 192.168.20.254 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/1
 ip address 198.51.100.1 255.255.255.252
 ip nat outside
 crypto map MY_MAP
 no shutdown

! --- Default WAN Route ---
ip route 0.0.0.0 0.0.0.0 198.51.100.2

! --- 1. NAT Bypass / Exemption ---
access-list 110 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.20.0 0.0.0.255 any
ip nat inside source list 110 interface GigabitEthernet0/1 overload

! --- 2. IPsec Phase 1 (ISAKMP) ---
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key SecretKey123 address 203.0.113.1

! --- 3. IPsec Phase 2 (Transform-Set & Interesting Traffic ACL) ---
crypto ipsec transform-set MY_SET esp-aes esp-sha-hmac
access-list 120 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255

! --- 4. Crypto Map Assembly & Binding ---
crypto map MY_MAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set MY_SET
 match address 120
```

---

## 🔍 Verification & Security Testing

### 1. VPN Tunnel Verification (Inter-site Ping)
From **PC1** (`192.168.10.1`), initiate an ICMP ping to **PC2** (`192.168.20.1`):
```bash
PC1> ping 192.168.20.1
84 bytes from 192.168.20.1 icmp_seq=1 ttl=62 time=12.450 ms
```

### 2. Phase 1 Verification (ISAKMP SA)
On **R1**, diagnostic commands confirm operational status for Phase 1:
```cisco
R1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
198.51.100.1    203.0.113.1     QM_IDLE           1001 ACTIVE
```
*The `QM_IDLE` state confirms successful IKE Phase 1 negotiation.*

### 3. IPsec Encryption Verification (Phase 2 SA)
On **R1**, inspect encapsulated and encrypted packet counters:
```cisco
R1# show crypto ipsec sa
    #pkts encaps: 25, #pkts encrypt: 25, #pkts digest: 25
    #pkts decaps: 25, #pkts decrypt: 25, #pkts verify: 25
```
*The `encrypt` and `decrypt` counters incrementing prove that traffic crosses the public Internet inside an encrypted tunnel.*

---

## 💡 Key Engineering Concepts Demonstrated
* **Diffie-Hellman Key Exchange (Group 2):** Secure generation of shared secret keys over insecure networks.
* **IKEv1 / IPsec Architecture:** Clear separation between control channel (Phase 1) and user data protection (Phase 2).
* **Cisco IOS Processing Order:** Evaluation hierarchy between NAT ACLs and Crypto Map interception.
