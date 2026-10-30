# Technical Documentation – EtherChannel Lab

## 1. Introduction

This document describes the technical implementation of an EtherChannel-based network infrastructure using Cisco Packet Tracer.

The lab focuses on link aggregation, a technology that combines multiple physical Ethernet links into a single logical interface.

Three EtherChannel configuration methods are implemented:

- PAgP (Port Aggregation Protocol).
- LACP (Link Aggregation Control Protocol).
- Static EtherChannel (`mode on`).

The infrastructure also integrates VLAN segmentation, Layer 2 trunking, Layer 3 switching, inter-VLAN routing, DHCP relay, and static routing.

## 2. Technical Objectives

The main objectives are:

- Understand the principles and benefits of EtherChannel.
- Configure EtherChannel using PAgP.
- Configure EtherChannel using LACP.
- Implement static EtherChannel without negotiation.
- Distinguish Layer 2 and Layer 3 EtherChannel configurations.
- Configure VLAN trunks over aggregated links.
- Implement inter-VLAN routing using multilayer switches.
- Establish routed connectivity between multilayer switches.
- Verify EtherChannel operation and end-to-end connectivity.

## 3. Network Architecture

The topology consists of four Cisco switches:

| Device | Role |
|---|---|
| MLS1 | Multilayer switch handling user VLANs and inter-VLAN routing |
| MLS2 | Multilayer switch handling the server VLAN |
| SW1 | Access switch for SALES, IT, and HR |
| SW2 | Access switch for the server network |

Three independent EtherChannel groups interconnect the switches.

### 3.1 EtherChannel Link Mapping

| EtherChannel | Devices | Member Interfaces | Protocol | Mode |
|---|---|---|---|---|
| Channel 1 | SW1 – MLS1 | Fa0/22-24 | PAgP | Desirable |
| Channel 1 | SW2 – MLS2 | Fa0/22-24 | LACP | Active |
| Channel 2 | MLS1 – MLS2 | Gi0/1-2 | Static | On |

The first two EtherChannels provide Layer 2 connectivity through trunk interfaces.

The third EtherChannel provides Layer 3 connectivity through a routed logical interface.

## 4. EtherChannel Implementation

### 4.1 PAgP – Port Aggregation Protocol

PAgP is a Cisco proprietary protocol used to negotiate the formation of an EtherChannel between compatible devices.

In this implementation, three FastEthernet interfaces are bundled between SW1 and MLS1.

Both devices use the desirable mode, which actively initiates PAgP negotiation.

**Configuration on SW1 and MLS1:**

```cisco
interface range f0/22-24
 channel-protocol pagp
 channel-group 1 mode desirable
```

The logical interface is configured as an 802.1Q trunk.

**Port-Channel configuration:**

```cisco
interface port-channel 1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 1001
```

The resulting Port-Channel carries VLANs 10, 20, and 30 between the access and multilayer switches.

### 4.2 LACP – Link Aggregation Control Protocol

LACP is an industry-standard link aggregation protocol defined by IEEE 802.1AX.

Unlike PAgP, LACP is supported by equipment from multiple network vendors.

In this lab, three FastEthernet interfaces are bundled between SW2 and MLS2.

Both devices use the active mode, which actively initiates LACP negotiation.

**Configuration on SW2 and MLS2:**

```cisco
interface range f0/22-24
 channel-protocol lacp
 channel-group 1 mode active
```

The resulting logical interface is configured as a trunk carrying VLAN 50.

```cisco
interface port-channel 1
 switchport mode trunk
 switchport nonegotiate
 switchport trunk allowed vlan 50
 switchport trunk native vlan 1001
```

### 4.3 Static EtherChannel – Layer 3

The third EtherChannel is configured between MLS1 and MLS2 using static mode.

Unlike PAgP and LACP, static EtherChannel does not use a negotiation protocol to establish the bundle.

The two physical GigabitEthernet interfaces are configured as routed interfaces and assigned to channel-group 2.

**Configuration on both multilayer switches:**

```cisco
interface range g0/1-2
 no switchport
 channel-group 2 mode on
```

An IP address is then assigned directly to the logical interface.

**MLS1:**

```cisco
interface port-channel 2
 ip address 10.0.0.1 255.255.255.252
```

**MLS2:**

```cisco
interface port-channel 2
 ip address 10.0.0.2 255.255.255.252
```

The transit network is:

| Parameter | Value |
|---|---|
| Network | 10.0.0.0/30 |
| MLS1 | 10.0.0.1 |
| MLS2 | 10.0.0.2 |
| Subnet Mask | 255.255.255.252 |

The resulting Port-Channel operates at Layer 3 and provides routed connectivity between the two multilayer switches.

## 5. VLAN and IP Addressing

Four VLANs are configured to logically segment the network.

| VLAN ID | VLAN Name | Network | Gateway |
|---|---|---|---|
| 10 | SALES | 172.16.10.0/24 | 172.16.10.254 |
| 20 | IT | 172.16.20.0/24 | 172.16.20.254 |
| 30 | HR | 172.16.30.0/24 | 172.16.30.254 |
| 50 | SERVERS | 172.16.50.0/24 | 172.16.50.254 |

MLS1 provides the default gateways for VLANs 10, 20, and 30.

MLS2 provides the default gateway for VLAN 50.

### 5.1 Inter-VLAN Routing

Inter-VLAN routing is implemented on MLS1 using Switched Virtual Interfaces (SVIs).

```cisco
ip routing

interface vlan 10
 ip address 172.16.10.254 255.255.255.0

interface vlan 20
 ip address 172.16.20.254 255.255.255.0

interface vlan 30
 ip address 172.16.30.254 255.255.255.0
```

IP routing is enabled to allow communication between the different VLANs.

### 5.2 DHCP Relay

The DHCP server is located in the server network.

MLS1 forwards DHCP requests from the user VLANs to the DHCP server using the `ip helper-address` command.

```cisco
interface vlan 10
 ip helper-address 172.16.50.2

interface vlan 20
 ip helper-address 172.16.50.2

interface vlan 30
 ip helper-address 172.16.50.2
```

This allows clients in different broadcast domains to obtain their IP configurations from the centralized DHCP server.

## 6. Static Routing

Static routing provides communication between the user networks connected to MLS1 and the server network connected to MLS2.

**MLS1:**

```cisco
ip route 172.16.50.0 255.255.255.0 10.0.0.2
```

**MLS2:**

```cisco
ip route 172.16.10.0 255.255.255.0 10.0.0.1
ip route 172.16.20.0 255.255.255.0 10.0.0.1
ip route 172.16.30.0 255.255.255.0 10.0.0.1
```

These routes use the Layer 3 EtherChannel transit network as the next hop.

## 7. EtherChannel Load Balancing

The switches are configured to use source and destination IP addresses for EtherChannel load distribution.

```cisco
port-channel load-balance src-dst-ip
```

This mechanism distributes traffic across the available physical links according to a hash calculated from the source and destination IP addresses.

It is important to note that EtherChannel load balancing does not split an individual flow across all member links. Traffic distribution is performed per flow according to the selected hashing algorithm.

## 8. Verification and Validation

Several Cisco IOS commands are used to validate the configuration.

### 8.1 EtherChannel Status

```cisco
show etherchannel summary
```

This command verifies:

- EtherChannel group numbers.
- Negotiation protocols.
- Port-Channel operational status.
- Physical interfaces successfully bundled.

The expected member interface flag is `(P)`, indicating that the interface is successfully participating in the EtherChannel.

The observed results confirm:

| Device | Port-Channel | Protocol | Status |
|---|---|---|---|
| SW1 | Po1 | PAgP | SU |
| MLS1 | Po1 | PAgP | SU |
| SW2 | Po1 | LACP | SU |
| MLS2 | Po1 | LACP | SU |
| MLS1 | Po2 | Static | RU |
| MLS2 | Po2 | Static | RU |

**Status flags:**

- `S`: Layer 2 Port-Channel.
- `R`: Layer 3 Port-Channel.
- `U`: Port-Channel in use.
- `P`: Physical interface successfully bundled.

### 8.2 Trunk Verification

```cisco
show interfaces trunk
```

This command confirms that the Layer 2 Port-Channels are operational as trunks and that the expected VLANs are allowed.

### 8.3 VLAN Verification

```cisco
show vlan brief
```

This command verifies VLAN creation and access-port assignments.

### 8.4 Interface Verification

```cisco
show ip interface brief
```

This command confirms the operational state and IP addressing of the logical and routed interfaces.

### 8.5 Routing Table Verification

```cisco
show ip route
```

The routing tables confirm the presence of directly connected networks and the configured static routes.

## 9. Functional Testing

Beyond configuration verification, several functional tests were performed.

| Test | Objective | Evidence |
|---|---|---|
| EtherChannel verification | Validate PAgP, LACP, and static aggregation | `shows.txt` |
| DHCP test | Verify dynamic IP address assignment | `dhcp-test.png` |
| DNS test | Verify DNS name resolution | `dns-test.png` |
| HTTP test | Verify access to the HTTP service | `http-test.png` |
| End-to-end connectivity | Validate communication across network segments | `end-to-end-connectivity.png` |

The simulation environment and associated test evidence are available in the `tests/` directory.

## 10. Key Technical Takeaways

This lab demonstrates several important networking concepts:

1. EtherChannel combines multiple physical links into one logical interface.
2. PAgP and LACP provide protocol-based negotiation for link aggregation.
3. Static EtherChannel requires consistent manual configuration on both endpoints.
4. Layer 2 EtherChannel can operate as an 802.1Q trunk.
5. Layer 3 EtherChannel can provide routed point-to-point connectivity.
6. The logical Port-Channel interface centralizes the configuration of the aggregated link.
7. Load balancing distributes traffic across member interfaces based on a hashing algorithm.
8. EtherChannel provides redundancy and can improve aggregate bandwidth.

## 11. Conclusion

This lab provided practical experience implementing EtherChannel in a segmented switched network.

By configuring PAgP, LACP, and static link aggregation, the project demonstrates different approaches to combining physical interfaces into logical connections.

The integration of VLANs, trunking, inter-VLAN routing, DHCP relay, and static routing further illustrates how EtherChannel can be deployed within a functional network infrastructure.

All configurations and verification evidence are available in the repository.


## Author

**C. Victorin Vodinou**

Networking | Systems Administration | Network Security

[GitHub Profile](https://github.com/Victorin13)
[LinkedIn Profile](https://www.linkedin.com/in/victorin-vodinou/)
Gmail Prifile: victorinvodinou@gmail.com

This project is part of my hands-on networking lab portfolio.