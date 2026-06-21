# Phase 6 Notes - SR-MPLS and SRv6 Coexistence

## Common Misconception

A common assumption after Phase 6 is that the lab was migrated completely from SR-MPLS to SRv6.

That is not what happened.

The final lab runs both Segment Routing data planes simultaneously:

* SR-MPLS (IPv4 transport)
* SRv6 (IPv6 transport)

This allows direct comparison between the two technologies while keeping the same VPN service.

---

## Phase 5 Architecture

Before Phase 6, the transport stack was:

CE1
↓
L3VPN
↓
VPNv4
↓
SR-MPLS
↓
IS-IS IPv4
↓
CE2

The SR-MPLS transport used Prefix-SIDs:

* R1 = 16001
* R2 = 16002
* R3 = 16003
* R4 = 16004

Traffic Engineering was implemented using an SR-TE policy:

R1 → R2 → R3 → R4

---

## What Was Added in Phase 6

Phase 6 introduced a parallel SRv6 transport.

New components:

* IPv6-enabled core links
* IPv6 loopbacks
* SRv6 locators
* IS-IS IPv6 address-family
* SRv6 service SIDs (uDT4 / End.DT4)

Example locator assignments:

* R1 = fcbb:bb00:1::/48
* R2 = fcbb:bb00:2::/48
* R3 = fcbb:bb00:3::/48
* R4 = fcbb:bb00:4::/48

---

## What Did Not Change

The customer-facing service remained unchanged:

VRF:

* CUST-A

Route Distinguisher:

* 100:1

Route Target:

* 100:1

Customer ASNs:

* CE1 = AS65001
* CE2 = AS65002

Customer Loopbacks:

* CE1 = 11.11.11.11/32
* CE2 = 22.22.22.22/32

The CE devices were not modified.

The customer continued to see the same VPN service.

---

## Key Configuration Added for SRv6

Enable SRv6 transport:

router isis CORE
address-family ipv6 unicast
segment-routing srv6
locator MAIN

Define SRv6 locator:

segment-routing
srv6
locators
locator MAIN
prefix fcbb:bb00:1::/48

Enable SRv6 VPN service:

router bgp 100
neighbor 4.4.4.4
address-family vpnv4 unicast
encapsulation-type srv6

vrf CUST-A
address-family ipv4 unicast
segment-routing srv6
locator MAIN
alloc mode per-vrf

---

## Final State

The final lab contains:

Transport Technologies:

* SR-MPLS
* SRv6

Services:

* VPNv4
* L3VPN

Protection:

* TI-LFA

Traffic Engineering:

* SR-TE

The lab demonstrates how the same customer L3VPN service can operate over either an MPLS-based Segment Routing transport or an IPv6-native Segment Routing transport.

This mirrors the migration path many service providers are evaluating today.


# Phase 6 Configuration Changes and Rollback

## Objective

Phase 6 introduces SRv6 transport and SRv6 VPN services while preserving the existing SR-MPLS infrastructure.

The goal is to demonstrate SR-MPLS and SRv6 coexistence without disrupting existing L3VPN services.

---

# Configuration Changes

## 1. Add IPv6 Loopbacks

Before:

```cisco
interface Loopback0
 ipv4 address 1.1.1.1 255.255.255.255
```

After:

```cisco
interface Loopback0
 ipv4 address 1.1.1.1 255.255.255.255
 ipv6 address fcbb:bb00:1::1/128
```

Purpose:

* SRv6 source address
* SRv6 locator reachability
* IPv6 IS-IS advertisement

---

## 2. Enable IPv6 on Core Links

Before:

```cisco
interface GigabitEthernet0/0/0/1
 ipv4 address 10.12.0.1 255.255.255.252
```

After:

```cisco
interface GigabitEthernet0/0/0/1
 ipv4 address 10.12.0.1 255.255.255.252
 ipv6 enable
```

Purpose:

* Enable IPv6 transport
* Support SRv6 forwarding

---

## 3. Enable IS-IS IPv6

Added:

```cisco
router isis CORE
 address-family ipv6 unicast
  metric-style wide
  segment-routing srv6
   locator MAIN
```

Purpose:

* Advertise SRv6 locators
* Build IPv6 underlay

---

## 4. Create SRv6 Locator

Added:

```cisco
segment-routing
 srv6
  encapsulation
   source-address fcbb:bb00:1::1
  !
  locators
   locator MAIN
    micro-segment behavior unode psp-usd
    prefix fcbb:bb00:1::/48
```

Purpose:

* Allocate SRv6 node SID
* Enable SRv6 encapsulation

Verification:

```bash
show segment-routing srv6 locator
show segment-routing srv6 sid
```

---

## 5. Enable SRv6 VPN Service

Added:

```cisco
router bgp 100
 neighbor 4.4.4.4
  address-family vpnv4 unicast
   encapsulation-type srv6
```

Purpose:

* Advertise SRv6 VPN information
* Replace VPN label signaling with SRv6 SID signaling

---

## 6. Enable Per-VRF SRv6 SID Allocation

Added:

```cisco
router bgp 100
 vrf CUST-A
  address-family ipv4 unicast
   segment-routing srv6
    locator MAIN
    alloc mode per-vrf
```

Purpose:

* Generate End.DT4 / uDT4 service SID
* Associate VPN service with VRF CUST-A

Verification:

```bash
show segment-routing srv6 sid
```

Expected:

* uN SID
* End.DT4 SID for VRF CUST-A

---

# Rollback Procedure

The rollback removes SRv6 and returns the lab to the Phase 5 state (SR-MPLS only).

---

## Step 1 - Remove SRv6 from VRF

```cisco
router bgp 100
 vrf CUST-A
  address-family ipv4 unicast
   no segment-routing srv6
```

---

## Step 2 - Remove SRv6 VPN Encapsulation

```cisco
router bgp 100
 neighbor 4.4.4.4
  address-family vpnv4 unicast
   no encapsulation-type srv6
```

---

## Step 3 - Remove IS-IS IPv6 SRv6

```cisco
router isis CORE
 no address-family ipv6 unicast
```

---

## Step 4 - Remove SRv6 Locator

```cisco
segment-routing
 no srv6
```

---

## Step 5 - Remove IPv6 from Core Interfaces

Example:

```cisco
interface GigabitEthernet0/0/0/1
 no ipv6 enable
```

Repeat on all core links.

---

## Step 6 - Remove IPv6 Loopback Address

```cisco
interface Loopback0
 no ipv6 address fcbb:bb00:1::1/128
```

---

# Expected Rollback Result

After rollback:

Available Features:

✓ IS-IS IPv4

✓ SR-MPLS

✓ Prefix-SIDs

✓ TI-LFA

✓ SR-TE

✓ VPNv4

✓ MPLS L3VPN

Removed Features:

✗ IPv6 Underlay

✗ SRv6 Locator

✗ SRv6 Node SID

✗ SRv6 End.DT4 Service SID

The lab returns to the completed Phase 5 architecture.
