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
