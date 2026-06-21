# IP Addressing and Parameter Planning

## Purpose

This document describes the addressing, identifiers, and parameter planning used throughout the SR-MPLS/SRv6 lab.

The goal is to maintain a predictable structure where router IDs, Prefix-SIDs, SRv6 locators, IS-IS NET IDs, and interface addressing all follow a consistent pattern.

---

# Device Roles

| Device | Role               |
| ------ | ------------------ |
| R1     | PE (Provider Edge) |
| R2     | P (Provider Core)  |
| R3     | P (Provider Core)  |
| R4     | PE (Provider Edge) |
| CE1    | Customer Edge      |
| CE2    | Customer Edge      |

---

# IPv4 Loopback Plan

Loopbacks are used for:

* Router IDs
* BGP update source
* IS-IS reachability
* SR-MPLS Prefix-SIDs

| Router | Loopback   |
| ------ | ---------- |
| R1     | 1.1.1.1/32 |
| R2     | 2.2.2.2/32 |
| R3     | 3.3.3.3/32 |
| R4     | 4.4.4.4/32 |

Design rule:

Router N uses N.N.N.N/32.

---

# Core IPv4 Addressing Plan

Core links follow the format:

10.XY.0.0/30

Where:

* X = first router number
* Y = second router number

| Link  | Network      |
| ----- | ------------ |
| R1-R2 | 10.12.0.0/30 |
| R1-R3 | 10.13.0.0/30 |
| R2-R3 | 10.23.0.0/30 |
| R2-R4 | 10.24.0.0/30 |
| R3-R4 | 10.34.0.0/30 |

This makes it easy to identify connected routers from the subnet.

Example:

10.24.0.0/30 immediately indicates a link between R2 and R4.

---

# IS-IS NET Planning

Each router uses a unique NET ID.

Format:

49.0001.0000.0000.000X.00

| Router | NET                       |
| ------ | ------------------------- |
| R1     | 49.0001.0000.0000.0001.00 |
| R2     | 49.0001.0000.0000.0002.00 |
| R3     | 49.0001.0000.0000.0003.00 |
| R4     | 49.0001.0000.0000.0004.00 |

The final digit matches the router number.

---

# SR-MPLS Prefix-SID Plan

Default SRGB:

16000-23999

Prefix-SID assignment:

| Router | Prefix-SID Index | Label |
| ------ | ---------------- | ----- |
| R1     | 1                | 16001 |
| R2     | 2                | 16002 |
| R3     | 3                | 16003 |
| R4     | 4                | 16004 |

Design rule:

Prefix-SID = 16000 + Router Number

This makes label identification simple during troubleshooting.

Example:

16004 immediately identifies R4.

---

# SRv6 Locator Plan

Each router receives a unique /48 locator.

Format:

fcbb:bb00:X::/48

Where:

X = Router Number

| Router | Locator          |
| ------ | ---------------- |
| R1     | fcbb:bb00:1::/48 |
| R2     | fcbb:bb00:2::/48 |
| R3     | fcbb:bb00:3::/48 |
| R4     | fcbb:bb00:4::/48 |

Loopback IPv6 addresses:

| Router | IPv6 Loopback      |
| ------ | ------------------ |
| R1     | fcbb:bb00:1::1/128 |
| R2     | fcbb:bb00:2::1/128 |
| R3     | fcbb:bb00:3::1/128 |
| R4     | fcbb:bb00:4::1/128 |

Design rule:

Locator number always matches router number.

---

# BGP Planning

Provider ASN:

100

Customer ASNs:

| Device | ASN   |
| ------ | ----- |
| CE1    | 65001 |
| CE2    | 65002 |

MP-BGP VPNv4 Peering:

| Peer    | Update Source |
| ------- | ------------- |
| R1 ↔ R4 | Loopback0     |

---

# VRF Planning

Customer Name:

CUST-A

| Parameter | Value |
| --------- | ----- |
| RD        | 100:1 |
| RT Import | 100:1 |
| RT Export | 100:1 |

Single VRF used throughout the lab.

---

# Customer Addressing

## CE1

| Interface | Address         |
| --------- | --------------- |
| Loopback0 | 11.11.11.11/32  |
| WAN       | 192.168.11.2/30 |

## CE2

| Interface | Address         |
| --------- | --------------- |
| Loopback0 | 22.22.22.22/32  |
| WAN       | 192.168.44.2/30 |

Provider-facing interfaces:

| Link   | Network         |
| ------ | --------------- |
| R1-CE1 | 192.168.11.0/30 |
| R4-CE2 | 192.168.44.0/30 |

---

# Traffic Engineering Planning

Default IS-IS behavior:

R1 → R4 uses ECMP through:

* R1 → R2 → R4
* R1 → R3 → R4

SR-TE Scenic Policy:

R1 → R2 → R3 → R4

Segment List:

* 16002
* 16003
* 16004

This intentionally overrides the shortest-path decision.

---

# Design Philosophy

The lab uses a consistent numbering scheme:

* Router ID = Router Number
* Prefix-SID = Router Number
* Locator = Router Number
* NET ID = Router Number

Examples:

R4:

* Router ID = 4.4.4.4
* Prefix-SID = 16004
* Locator = fcbb:bb00:4::/48
* NET = 49.0001.0000.0000.0004.00

This simplifies troubleshooting and reduces operational complexity.
