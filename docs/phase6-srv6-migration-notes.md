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
