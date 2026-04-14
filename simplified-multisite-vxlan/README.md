# Simplified Multi-Site VXLAN with IPsec Tunnel

Two-site VXLAN fabric connected via an IPsec SVTI tunnel between two Cisco CSR1000v routers. Each site has a single spine and leaf (NX-OSv 9000) with OSPF underlay and BGP EVPN, extending VLAN 10 / VNI 100010 across sites over a secure WAN link.

---

## Topology Diagram

```
          SITE A                                           SITE B

  PC-A ─── LEAF-A ─── SPINE-A ─── CSR-A ═══IPsec═══ CSR-B ─── SPINE-B ─── LEAF-B ─── PC-B
 (Eth1/8) (Eth1/1)  (Eth1/9) (Gi2)  (Gi1─Gi1)  (Gi2) (Eth1/9)  (Eth1/1) (Eth1/8)
```

### Physical Cabling

| Connection             | Side A Interface     | Side B Interface     |
|------------------------|----------------------|----------------------|
| Router-to-Router (WAN) | CSR-A Gi1            | CSR-B Gi1            |
| Router-to-Spine        | CSR-A Gi2            | Spine-A Eth1/9       |
| Router-to-Spine        | CSR-B Gi2            | Spine-B Eth1/9       |
| Spine-to-Leaf          | Spine-A Eth1/1       | Leaf-A Eth1/1        |
| Spine-to-Leaf          | Spine-B Eth1/1       | Leaf-B Eth1/1        |
| Leaf-to-PC             | Leaf-A Eth1/8        | PC-A                 |
| Leaf-to-PC             | Leaf-B Eth1/8        | PC-B                 |

---

## IP Address Table

### Site A

| Device  | Interface        | IP Address          | Description              |
|---------|-----------------|---------------------|--------------------------|
| CSR-A   | Loopback0        | 10.0.0.100/32       | BGP router-id            |
| CSR-A   | GigabitEthernet1 | 100.0.0.1/30        | WAN (tunnel source)      |
| CSR-A   | GigabitEthernet2 | 10.99.0.1/24        | Fabric link to Spine-A   |
| CSR-A   | Tunnel100        | 172.16.100.1/30     | IPsec SVTI to CSR-B      |
| SPINE-A | loopback0        | 10.0.0.1/32         | BGP router-id            |
| SPINE-A | Ethernet1/1      | 10.1.1.1/24         | Link to Leaf-A           |
| SPINE-A | Ethernet1/9      | 10.99.0.2/24        | Link to CSR-A            |
| LEAF-A  | loopback0        | 10.0.0.11/32        | BGP router-id / VTEP     |
| LEAF-A  | Ethernet1/1      | 10.1.1.11/24        | Link to Spine-A          |
| LEAF-A  | Ethernet1/8      | switchport vlan 10  | Access port to PC-A      |
| PC-A    | eth0             | 192.168.10.1/24     | Tenant host              |

### Site B

| Device  | Interface        | IP Address          | Description              |
|---------|-----------------|---------------------|--------------------------|
| CSR-B   | Loopback0        | 10.0.0.200/32       | BGP router-id            |
| CSR-B   | GigabitEthernet1 | 100.0.0.2/30        | WAN (tunnel source)      |
| CSR-B   | GigabitEthernet2 | 10.99.1.1/24        | Fabric link to Spine-B   |
| CSR-B   | Tunnel100        | 172.16.100.2/30     | IPsec SVTI to CSR-A      |
| SPINE-B | loopback0        | 10.0.0.201/32       | BGP router-id            |
| SPINE-B | Ethernet1/1      | 10.3.1.1/24         | Link to Leaf-B           |
| SPINE-B | Ethernet1/9      | 10.99.1.2/24        | Link to CSR-B            |
| LEAF-B  | loopback0        | 10.0.0.211/32       | BGP router-id / VTEP     |
| LEAF-B  | Ethernet1/1      | 10.3.1.11/24        | Link to Spine-B          |
| LEAF-B  | Ethernet1/8      | switchport vlan 10  | Access port to PC-B      |
| PC-B    | eth0             | 192.168.10.2/24     | Tenant host              |

---

## VNI / VLAN Mapping

| VLAN | VNI    | Description      | Devices                |
|------|--------|------------------|------------------------|
| 10   | 100010 | Tenant network   | LEAF-A, LEAF-B         |

VNI 100010 is identical on both sites so that VXLAN frames are decoded without translation.

---

## PC Configuration (GNS3 VPCS)

```
# PC-A
ip 192.168.10.1 255.255.255.0

# PC-B
ip 192.168.10.2 255.255.255.0
```

No default gateway is needed — both PCs are in the same Layer 2 domain (VLAN 10 / VNI 100010) extended across the VXLAN fabric.

---

## Key Design Decisions

### Single BGP AS (iBGP)
Both sites share **AS 65000**. All BGP EVPN peerings are iBGP. The CSR routers and spines act as route reflectors to propagate EVPN MAC/IP routes across the full chain.

### EVPN Route Reflector Chain
```
LEAF-A ↔ SPINE-A (RR) ↔ CSR-A (RR) ↔ CSR-B (RR) ↔ SPINE-B (RR) ↔ LEAF-B
```
Each hop acts as a route reflector, ensuring EVPN routes originated at one leaf reach the other leaf across both sites.

### OSPF Area 0 as Shared Underlay
OSPF area 0 runs on all fabric links **and** across the IPsec tunnel (`172.16.100.0/30`). This makes all loopback addresses (VTEPs, router-ids) reachable end-to-end, which is required for:
- BGP EVPN sessions (use loopback as update-source)
- VXLAN data plane (NVE source-interface is the loopback)

### IPsec SVTI (Static Virtual Tunnel Interface)
The CSR1000v routers use IPsec SVTI (not GRE over IPsec):
- Routable point-to-point interface (`Tunnel100`)
- Native IPsec encryption without GRE overhead
- IKEv2 with AES-256-CBC / SHA-256 / DH Group 14
- Pre-shared key: `Cisco123!`

### MTU Considerations
VXLAN adds ~50 bytes of overhead. IPsec adds ~70 bytes (ESP header/trailer/ICV).
- NX-OS fabric links: MTU 9150 (jumbo frames)
- CSR fabric-facing Gi2: MTU 9000
- CSR Tunnel100: `ip mtu 1400` and `ip tcp adjust-mss 1360`
- For full 1500-byte PC frames over the tunnel, the WAN path should support at least ~1600-byte frames, or accept IP fragmentation on the CSR

---

## Folder Structure

```
simplified-multisite-vxlan/
├── README.md
├── site-a/
│   ├── csr-a.txt       CSR1000v Site A (IOS-XE: IPsec + OSPF + BGP EVPN RR)
│   ├── spine-a.txt     SPINE-A NX-OSv 9000 (OSPF + BGP EVPN RR)
│   └── leaf-a.txt      LEAF-A NX-OSv 9000 (OSPF + BGP EVPN + VXLAN VNI 100010)
└── site-b/
    ├── csr-b.txt       CSR1000v Site B (IOS-XE: IPsec + OSPF + BGP EVPN RR)
    ├── spine-b.txt     SPINE-B NX-OSv 9000 (OSPF + BGP EVPN RR)
    └── leaf-b.txt      LEAF-B NX-OSv 9000 (OSPF + BGP EVPN + VXLAN VNI 100010)
```

---

## Verification Commands

### IOS-XE (CSR-A / CSR-B)

```ios
! Verify IPsec tunnel is up
show crypto ikev2 sa
show crypto ipsec sa

! Verify OSPF neighbors (should see spine + remote CSR via tunnel)
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

! Check VXLAN NVE peer table (should show remote VTEP from other site)
show nve peers

! Verify MAC/IP routes in EVPN table
show bgp l2vpn evpn

! Check MAC address table
show mac address-table

! Verify OSPF adjacencies
show ip ospf neighbors
```

### End-to-End Test

1. From **PC-A**: `ping 192.168.10.2`
2. On **LEAF-A**: `show nve peers` → should show 10.0.0.211 (LEAF-B VTEP)
3. On **CSR-A**: `show crypto ipsec sa` → encrypted/decrypted counters should increment
4. On **LEAF-B**: `show mac address-table` → should show PC-A MAC learned via VXLAN
