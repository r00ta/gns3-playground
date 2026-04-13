# CLOS Multi-Site VXLAN with IPsec Tunnel

Two-site VXLAN fabric connected via an IPsec SVTI tunnel between two Cisco CSR1000v routers. Both sites run NX-OS spines and leaves with OSPF underlay and BGP EVPN, extending the same VNIs across sites over a secure WAN link.

---

## Topology Diagram

```
                          SITE A                                      SITE B
 ┌──────────────────────────────────────────┐   ┌──────────────────────────────────────┐
 │                                          │   │                                      │
 │  SPINE1 (10.0.0.1)  SPINE2 (10.0.0.2)  │   │  SPINE1-B (10.0.0.201)              │
 │       │  \            /  │              │   │       │  \                           │
 │       │   \          /   │              │   │       │   \  SPINE2-B (10.0.0.202)  │
 │       │    \        /    │              │   │       │    \      │                  │
 │   LEAF1   LEAF2   LEAF3  │              │   │   LEAF1-B   LEAF2-B                 │
 │ (10.0.0.11)(10.0.0.12)(10.0.0.13)      │   │ (10.0.0.211)(10.0.0.212)            │
 │                          │              │   │                   │                  │
 │               CSR-A ─────┘              │   │            CSR-B ─┘                  │
 │          (lo: 10.0.0.100)               │   │       (lo: 10.0.0.200)               │
 │          (WAN: 100.0.0.1) ─────IPsec────┼───┼────── (WAN: 100.0.0.2)              │
 │                           Tunnel100     │   │       Tunnel100                      │
 │                    172.16.100.1/30      │   │       172.16.100.2/30                │
 └──────────────────────────────────────────┘   └──────────────────────────────────────┘
```

---

## IP Address Table

### Site A

| Device   | Interface        | IP Address          | Description              |
|----------|-----------------|---------------------|--------------------------|
| SPINE1   | loopback0        | 10.0.0.1/32         | BGP router-id / VTEP     |
| SPINE1   | Ethernet1/1      | 10.1.1.1/24         | Link to LEAF1            |
| SPINE1   | Ethernet1/2      | 10.1.2.1/24         | Link to LEAF2            |
| SPINE1   | Ethernet1/3      | 10.1.3.1/24         | Link to LEAF3            |
| SPINE1   | Ethernet1/4      | 10.99.0.2/24        | Link to CSR-A fabric     |
| SPINE2   | loopback0        | 10.0.0.2/32         | BGP router-id / VTEP     |
| SPINE2   | Ethernet1/1      | 10.2.1.2/24         | Link to LEAF1            |
| SPINE2   | Ethernet1/2      | 10.2.2.2/24         | Link to LEAF2            |
| SPINE2   | Ethernet1/3      | 10.2.3.2/24         | Link to LEAF3            |
| SPINE2   | Ethernet1/4      | 10.99.0.3/24        | Link to CSR-A fabric     |
| LEAF1    | loopback0        | 10.0.0.11/32        | BGP router-id / VTEP     |
| LEAF1    | Ethernet1/1      | 10.1.1.11/24        | Link to SPINE1           |
| LEAF1    | Ethernet1/2      | 10.2.1.11/24        | Link to SPINE2           |
| LEAF2    | loopback0        | 10.0.0.12/32        | BGP router-id / VTEP     |
| LEAF2    | Ethernet1/1      | 10.1.2.12/24        | Link to SPINE1           |
| LEAF2    | Ethernet1/2      | 10.2.2.12/24        | Link to SPINE2           |
| LEAF3    | loopback0        | 10.0.0.13/32        | BGP router-id / VTEP     |
| LEAF3    | Ethernet1/1      | 10.1.3.13/24        | Link to SPINE1           |
| LEAF3    | Ethernet1/2      | 10.2.3.13/24        | Link to SPINE2           |
| CSR-A    | Loopback0        | 10.0.0.100/32       | BGP router-id            |
| CSR-A    | GigabitEthernet1 | 100.0.0.1/30        | WAN (tunnel source)      |
| CSR-A    | GigabitEthernet2 | 10.99.0.1/24        | Fabric-facing link       |
| CSR-A    | Tunnel100        | 172.16.100.1/30     | IPsec SVTI to CSR-B      |

### Site B

| Device    | Interface        | IP Address          | Description              |
|-----------|-----------------|---------------------|--------------------------|
| SPINE1-B  | loopback0        | 10.0.0.201/32       | BGP router-id / VTEP     |
| SPINE1-B  | Ethernet1/1      | 10.1.1.201/24       | Link to LEAF1-B          |
| SPINE1-B  | Ethernet1/2      | 10.1.2.201/24       | Link to LEAF2-B          |
| SPINE1-B  | Ethernet1/3      | 10.99.1.2/24        | Link to CSR-B fabric     |
| SPINE2-B  | loopback0        | 10.0.0.202/32       | BGP router-id / VTEP     |
| SPINE2-B  | Ethernet1/1      | 10.2.1.202/24       | Link to LEAF1-B          |
| SPINE2-B  | Ethernet1/2      | 10.2.2.202/24       | Link to LEAF2-B          |
| SPINE2-B  | Ethernet1/3      | 10.99.1.3/24        | Link to CSR-B fabric     |
| LEAF1-B   | loopback0        | 10.0.0.211/32       | BGP router-id / VTEP     |
| LEAF1-B   | Ethernet1/1      | 10.1.1.211/24       | Link to SPINE1-B         |
| LEAF1-B   | Ethernet1/2      | 10.2.1.211/24       | Link to SPINE2-B         |
| LEAF2-B   | loopback0        | 10.0.0.212/32       | BGP router-id / VTEP     |
| LEAF2-B   | Ethernet1/1      | 10.1.2.212/24       | Link to SPINE1-B         |
| LEAF2-B   | Ethernet1/2      | 10.2.2.212/24       | Link to SPINE2-B         |
| CSR-B     | Loopback0        | 10.0.0.200/32       | BGP router-id            |
| CSR-B     | GigabitEthernet1 | 100.0.0.2/30        | WAN (tunnel source)      |
| CSR-B     | GigabitEthernet2 | 10.99.1.1/24        | Fabric-facing link       |
| CSR-B     | Tunnel100        | 172.16.100.2/30     | IPsec SVTI to CSR-A      |

---

## VNI / VLAN Mapping

| VLAN | VNI    | Description           | Site A Devices       | Site B Devices       |
|------|--------|-----------------------|----------------------|----------------------|
| 10   | 100010 | Tenant network 1      | LEAF1, LEAF2         | LEAF1-B, LEAF2-B     |
| 20   | 100020 | Tenant network 2      | LEAF2, LEAF3         | LEAF2-B              |

> VNI numbers are **identical** on both sites. This is required for seamless L2 extension across the IPsec tunnel — a VXLAN frame encapsulated with VNI 100010 at Site A is decoded correctly at Site B without any translation.

---

## Key Design Decisions

### Single BGP AS (iBGP)
Both sites run in **BGP AS 65000**. This simplifies the design: all BGP EVPN peerings are iBGP, and the CSR routers act as inter-site route reflectors, redistributing EVPN MAC/IP routes between Site A spines and Site B spines.

### Identical VNI Numbers Across Sites
VNI values (100010, 100020) are the same on both sites. This enables true Layer 2 extension — a MAC learned at LEAF1 (Site A) with VNI 100010 can be reached directly by LEAF1-B (Site B) using the same VNI.

### OSPF as Shared Underlay
OSPF area 0 is extended across the IPsec tunnel (network `172.16.100.0/30`). This makes all loopback addresses reachable from both sites, allowing BGP EVPN sessions to use loopbacks as update sources.

### IPsec SVTI (Static Virtual Tunnel Interface)
The CSR1000v routers use IPsec SVTI (not GRE over IPsec) for the WAN tunnel. SVTI provides:
- A routable point-to-point interface (`Tunnel100`)
- Native IPsec encryption without GRE overhead
- Clean MTU control via `ip mtu 1400`
- IKEv2 with AES-256 / SHA-256 / DH Group 14

### MTU and Fragmentation
VXLAN adds **~50 bytes** of overhead (8 VXLAN + 8 UDP + 20 outer IP + 14 outer Ethernet). IPsec adds another **~70 bytes** (ESP header, trailer, ICV). Total overhead can exceed 120 bytes.

- Physical MTU on all fabric links should be **≥ 1600** bytes (NX-OS links are set to 9150, so this is not a concern there)
- The CSR WAN/tunnel interface uses `ip mtu 1400` and `ip tcp adjust-mss 1360` to clamp TCP MSS and avoid fragmentation for TCP traffic
- For non-TCP traffic (UDP, ICMP), ensure the WAN provider supports at least 1500-byte frames, or enable `ip tcp adjust-mss` and consider PMTUD

---

## Folder Structure

```
clos-multisite-vxlan/
├── README.md
├── site-a/
│   ├── csr-a.txt       CSR1000v Site A (IOS-XE)
│   ├── spine01.txt     SPINE1 NX-OS (updated with CSR-A BGP peer)
│   ├── spine02.txt     SPINE2 NX-OS (updated with CSR-A BGP peer)
│   ├── leaf01.txt      LEAF1 NX-OS (VNI 100010)
│   ├── leaf02.txt      LEAF2 NX-OS (VNI 100010 + 100020)
│   └── leaf03.txt      LEAF3 NX-OS (VNI 100020)
└── site-b/
    ├── csr-b.txt       CSR1000v Site B (IOS-XE)
    ├── spine01.txt     SPINE1-B NX-OS
    ├── spine02.txt     SPINE2-B NX-OS
    ├── leaf01.txt      LEAF1-B NX-OS (VNI 100010)
    └── leaf02.txt      LEAF2-B NX-OS (VNI 100010 + 100020)
```

---

## Verification Commands

### IOS-XE (CSR-A / CSR-B)

```ios
! Verify IPsec tunnel is up
show crypto ikev2 sa
show crypto ipsec sa

! Verify OSPF neighbors (should see spines and the remote CSR)
show ip ospf neighbor

! Verify BGP EVPN sessions
show bgp l2vpn evpn summary

! Inspect EVPN routes received
show bgp l2vpn evpn

! Check routing table for remote loopbacks
show ip route 10.0.0.0 255.0.0.0 longer-prefixes
```

### NX-OS (Spines / Leaves)

```nxos
! Verify BGP EVPN sessions
show bgp l2vpn evpn summary

! Check VXLAN NVE peer table (should show remote VTEPs from both sites)
show nve peers

! Verify MAC/IP routes in EVPN table
show bgp l2vpn evpn

! Check MAC address table
show mac address-table

! Verify OSPF adjacencies
show ip ospf neighbors
```

### End-to-end Test

1. Ping from a server in VLAN 10 at Site A to a server in VLAN 10 at Site B
2. On the leaf where traffic originates, verify: `show nve peers` shows the remote VTEP
3. On CSR-A/CSR-B: `show crypto ipsec sa` shows encrypted/decrypted packet counters incrementing
