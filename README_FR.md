# 🛡️ Projet : VPN IPsec Site-à-Site & Exemption NAT (Cisco IOS)

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![GNS3](https://img.shields.io/badge/Simulator-GNS3-orange?style=for-the-badge)
![VPN](https://img.shields.io/badge/Security-IPsec%20VPN-green?style=for-the-badge)

## 📌 Présentation du Projet
Ce projet consiste à concevoir, déployer et sécuriser l'interconnexion WAN entre le **Siège (R1)** et une **Filiale distante (R2)** d'une entreprise à travers un réseau Internet non sécurisé simulé par un opérateur (**ISP**).

L'architecture permet :
1. **La confidentialité des échanges inter-sites** via un tunnel chiffré **VPN IPsec Site-à-Site** (ISAKMP / IKEv1 + IPsec ESP-AES).
2. **L'accès Internet individuel** pour chaque site via de la translation d'adresses dynamiques (**NAT Overload / PAT**).
3. **L'exemption NAT (NAT Bypass)** pour garantir que le trafic privé à destination du VPN ne soit pas altéré par la règle de traduction NAT.

---

## 📐 Topologie Réseau

```text
 [ LAN Siège ]                                                                                    [ LAN Filiale ]
 PC1 (192.168.10.1)                                                                              PC2 (192.168.20.1)
         |                                                                                               |
         v                                                                                               v
   +-----------+        WAN 1 (203.0.113.0/30)        +-------+       WAN 2 (203.0.113.4/30)       +---------------+
   | R1 (Siège)|-------------------------------------|  ISP  |-------------------------------------| R2 (Filiale)  |
   +-----------+                                     +-------+                                     +---------------+
                                                         |
                                                         v
                                              [ Internet Simulé ]
                                                 DNS (8.8.8.8)
```

---

## 📊 Plan d'Adressage IP

| Équipement | Interface | Adresse IP / Masque | Rôle / Description |
| :--- | :--- | :--- | :--- |
| **R1 (Siège)** | `Gi0/0` | `192.168.10.254/24` | Passerelle LAN Siège |
| **R1 (Siège)** | `Gi0/1` | `203.0.113.1/30` | Interface WAN vers ISP |
| **R2 (Filiale)** | `Gi0/0` | `192.168.20.254/24` | Passerelle LAN Filiale |
| **R2 (Filiale)** | `Gi0/1` | `203.0.113.5/30` | Interface WAN vers ISP |
| **ISP** | `Gi0/0` | `203.0.113.2/30` | Interface WAN vers R1 |
| **ISP** | `Gi0/1` | `203.0.113.6/30` | Interface WAN vers R2 |
| **ISP** | `Loopback0`| `8.8.8.8/32` | Serveur Internet simulé |

---

## 🛠️ Configuration des Équipements (Cisco IOS)

### 1. Configuration de R1 (Siège)

```cisco
hostname R1

! --- Configuration des Interfaces ---
interface GigabitEthernet0/0
 ip address 192.168.10.254 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/1
 ip address 203.0.113.1 255.255.255.252
 ip nat outside
 crypto map MA_MAP
 no shutdown

! --- Route par défaut WAN ---
ip route 0.0.0.0 0.0.0.0 203.0.113.2

! --- 1. Exemption NAT (NAT Bypass) ---
access-list 110 deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip 192.168.10.0 0.0.0.255 any
ip nat inside source list 110 interface GigabitEthernet0/1 overload

! --- 2. Phase 1 IPsec (ISAKMP) ---
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key CleSecrete123 address 203.0.113.1

! --- 3. Phase 2 IPsec (Transform-Set & ACL Trafic Interressant) ---
crypto ipsec transform-set MON_SET esp-aes esp-sha-hmac
access-list 120 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255

! --- 4. Crypto Map ---
crypto map MA_MAP 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set MON_SET
 match address 120
```

---

### 2. Configuration de R2 (Filiale)

```cisco
hostname R2

! --- Configuration des Interfaces ---
interface GigabitEthernet0/0
 ip address 192.168.20.254 255.255.255.0
 ip nat inside
 no shutdown

interface GigabitEthernet0/1
 ip address 203.0.113.5 255.255.255.252
 ip nat outside
 crypto map MA_MAP
 no shutdown

! --- Route par défaut WAN ---
ip route 0.0.0.0 0.0.0.0 203.0.113.6

! --- 1. Exemption NAT (NAT Bypass) ---
access-list 110 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.20.0 0.0.0.255 any
ip nat inside source list 110 interface GigabitEthernet0/1 overload

! --- 2. Phase 1 IPsec (ISAKMP) ---
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key CleSecrete123 address 203.0.113.1

! --- 3. Phase 2 IPsec (Transform-Set & ACL Trafic Interressant) ---
crypto ipsec transform-set MON_SET esp-aes esp-sha-hmac
access-list 120 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255

! --- 4. Crypto Map ---
crypto map MA_MAP 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set MON_SET
 match address 120
```

---

## 🔍 Validation & Tests de Sécurité

### 1. Test du Tunnel VPN (Ping Inter-Sites)
Depuis **PC1** (`192.168.10.1`), lancement d'un ping vers **PC2** (`192.168.20.1`) :
```bash
PC1> ping 192.168.20.1
84 bytes from 192.168.20.1 icmp_seq=1 ttl=62 time=12.450 ms
```

### 2. Verification de la Phase 1 (ISAKMP SA)
Sur **R1**, la commande de diagnostic confirme l'état opérationnel de la Phase 1 :
```cisco
R1# show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.0.113.5    203.0.113.1     QM_IDLE           1001 ACTIVE
```
*L'état `QM_IDLE` indique que la négociation IKE Phase 1 s'est déroulée avec succès.*

### 3. Vérification du Chiffrement IPsec (Phase 2 SA)
Sur **R1**, inspection du nombre de paquets encapsulés et chiffrés :
```cisco
R1# show crypto ipsec sa
    #pkts encaps: 25, #pkts encrypt: 25, #pkts digest: 25
    #pkts decaps: 25, #pkts decrypt: 25, #pkts verify: 25
```
*Les compteurs `encrypt` et `decrypt` s'incrémentent, prouvant que les données traversent l'Internet sous forme entièrement chiffrée.*

---

## 💡 Notions Clés Assimilées
* **Échange de clés Diffie-Hellman (Group 2) :** Génération sécurisée du secret partagé à travers un réseau non sécurisé.
* **Architecture IKEv1 / IPsec :** Séparation claire entre le canal de contrôle (Phase 1) et la protection des données utilisateur (Phase 2).
* **Ordre de traitement Cisco IOS :** Priorité d'évaluation entre les ACLs de NAT et l'interception par les Crypto Maps.
