# PE Configuration Template (Cisco IOS XR)

A fill-in-the-blanks PE template covering the dual-stack underlay (SR-MPLS + SRv6), an
L3VPN, and an EVPN-VPWS L2VPN. **The lesson is in what's a variable and what isn't:**
every `{{VARIABLE}}` is a per-device or per-service value; everything in plain text is
**domain-wide and never changes**. That sorting is the understanding.

## Fill-in table

| Variable | Scope | R1 | R4 |
|---|---|---|---|
| `{{HOSTNAME}}` | per-device | R1 | R4 |
| `{{V4_LOOPBACK}}` | per-device | 1.1.1.1 | 4.4.4.4 |
| `{{V6_LOOPBACK}}` | per-device | fcbb:bb00:1::1 | fcbb:bb00:4::1 |
| `{{SID_INDEX}}` | per-device (unique) | 1 | 4 |
| `{{LOCATOR_PREFIX}}` | per-device (shared block) | fcbb:bb00:1::/48 | fcbb:bb00:4::/48 |
| `{{NET}}` | per-device (unique sys-id) | 49.0001.0000.0000.0001.00 | 49.0001.0000.0000.0004.00 |
| `{{REMOTE_PE}}` | per-device | 4.4.4.4 | 1.1.1.1 |
| `{{L3_CE_INTF}}` / `{{L3_CE_IP}}` | per-device | Gi0/0/0/0 / 192.168.11.1/30 | Gi0/0/0/1 / 192.168.44.1/30 |
| `{{CE_PEER}}` / `{{CE_AS}}` | per-device | 192.168.11.2 / 65001 | 192.168.44.2 / 65002 |
| `{{L2_AC_INTF}}` | per-device | Gi0/0/0/2 | Gi0/0/0/0 |
| `{{RD}}` / `{{RT}}` | per-service | 100:1 / 100:1 | 100:1 / 100:1 |
| `{{EVI}}` / `{{SERVICE_ID}}` | per-service | 200 / 1 | 200 / 1 |

Domain-wide constants (same on every node, never variables): IS-IS process `CORE`,
`is-type level-2-only`, `metric-style wide`, `segment-routing mpls`, SRGB 16000–23999,
uSID behavior `unode psp-usd`, provider AS `100`.

---

## Template

```
hostname {{HOSTNAME}}
!
vrf CUST-A
 address-family ipv4 unicast
  import route-target
   {{RT}}
  !
  export route-target
   {{RT}}
  !
 !
!
route-policy PASS
  pass
end-policy
!
segment-routing
 srv6
  encapsulation
   source-address {{V6_LOOPBACK}}
  !
  locators
   locator MAIN
    micro-segment behavior unode psp-usd
    prefix {{LOCATOR_PREFIX}}
   !
  !
 !
!
evpn
 segment-routing srv6
  locator MAIN
 !
!
interface Loopback0
 ipv4 address {{V4_LOOPBACK}} 255.255.255.255
 ipv6 address {{V6_LOOPBACK}}/128
!
interface {{L3_CE_INTF}}
 vrf CUST-A
 ipv4 address {{L3_CE_IP}}
!
interface {{L2_AC_INTF}}
 l2transport
!
! --- core interfaces: ipv4 + ipv6 enable + IS-IS (repeat per core link) ---
!
router isis CORE
 is-type level-2-only
 net {{NET}}
 address-family ipv4 unicast
  metric-style wide
  segment-routing mpls
 !
 address-family ipv6 unicast
  metric-style wide
  segment-routing srv6
   locator MAIN
  !
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index {{SID_INDEX}}
  !
  address-family ipv6 unicast
  !
 !
 ! per core interface:
 !  point-to-point / address-family ipv4 unicast (+ ti-lfa) / address-family ipv6 unicast
!
router bgp 100
 bgp router-id {{V4_LOOPBACK}}
 address-family vpnv4 unicast
 !
 address-family l2vpn evpn
 !
 neighbor {{REMOTE_PE}}
  remote-as 100
  update-source Loopback0
  address-family vpnv4 unicast
   encapsulation-type srv6        ! omit this line to run the L3VPN over SR-MPLS
  !
  address-family l2vpn evpn
  !
 !
 vrf CUST-A
  rd {{RD}}
  address-family ipv4 unicast
   redistribute connected
   segment-routing srv6           ! omit this block to run the L3VPN over SR-MPLS
    locator MAIN
    alloc mode per-vrf
   !
  !
  neighbor {{CE_PEER}}
   remote-as {{CE_AS}}
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
!
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface {{L2_AC_INTF}}
   neighbor evpn evi {{EVI}} service {{SERVICE_ID}} segment-routing srv6
   ! SR-MPLS form instead: neighbor evpn evi {{EVI}} target <remote-ac> source <local-ac>
  !
 !
!
commit
```

---

## How to read this as a learning tool

1. Everything in **plain text** is domain-wide — it is the same on R1, R4, and any PE
   you ever add. If you understand *why* it's fixed, you understand the protocol.
2. Every `{{VARIABLE}}` is something you must decide per device or per service. The
   fill-in table tells you which.
3. The two inline comments show the **transport switch**: keep the SRv6 lines for SRv6,
   delete them for SR-MPLS. That tiny delta is the whole "transport-agnostic" idea.

Re-deriving this template from memory — and being able to say which bucket each value
falls in — is a strong test that the concepts have stuck.
