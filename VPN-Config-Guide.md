# VPN Configuration Guide — L2 & L3 VPN over SR-MPLS and SRv6

A quick-reference for the four service/transport combinations built in this lab, on
Cisco IOS XR. Examples use this lab's values (PEs R1 `1.1.1.1` / R4 `4.4.4.4`,
provider AS 100, locator `MAIN`); adapt as needed. Configs show one PE — the other
mirrors it.

**The pattern to remember:** the *service* config (VRF, EVI, CE side) is almost
identical regardless of transport. Switching SR-MPLS ↔ SRv6 is a small delta. The
core never changes.

| Service | Transport | Service SID | Delta vs SR-MPLS baseline |
|---|---|---|---|
| L3VPN (VPNv4) | SR-MPLS | MPLS VPN label | — (baseline) |
| L3VPN (VPNv4) | SRv6 | `uDT4` | `encapsulation-type srv6` + per-VRF SRv6 locator |
| L2VPN VPWS | SR-MPLS | PW label | `l2vpn evpn` AF + xconnect `target/source` |
| L2VPN VPWS | SRv6 | `uDX2` | `evpn` SRv6 locator + xconnect `service … segment-routing srv6` |

---

## Underlay prerequisites

**SR-MPLS** (per node): IS-IS Level-2 with `metric-style wide`, `segment-routing mpls`
under the IPv4 AF, and a `prefix-sid index N` on Loopback0. No LDP/RSVP.

**SRv6** (per node): an IPv6 loopback, `ipv6 enable` on core links, an IS-IS IPv6 AF
advertising the locator, and the global SRv6 block:
```
segment-routing
 srv6
  encapsulation
   source-address <node-ipv6-loopback>
  locators
   locator MAIN
    micro-segment behavior unode psp-usd
    prefix <node>::/48
```
> On XRv9000: `hw-module profile segment-routing srv6 mode micro-segment format f3216`
> then reload, if SRv6 SIDs don't appear.

Both underlays can run at the same time (dual-stack IGP) — a service picks one.

---

## Part 1 — L3VPN (MP-BGP VPNv4)

### Common base (both transports)

```
vrf CUST-A
 address-family ipv4 unicast
  import route-target
   100:1
  !
  export route-target
   100:1
  !
 !
!
route-policy PASS
  pass
end-policy
!
interface GigabitEthernet0/0/0/0      ! CE-facing, in the VRF
 vrf CUST-A
 ipv4 address 192.168.11.1 255.255.255.252
!
router bgp 100
 address-family vpnv4 unicast
 !
 neighbor 4.4.4.4                     ! remote PE loopback
  remote-as 100
  update-source Loopback0
  address-family vpnv4 unicast
  !
 !
 vrf CUST-A
  rd 100:1
  address-family ipv4 unicast
   redistribute connected
  !
  neighbor 192.168.11.2               ! the CE
   remote-as 65001
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
!
```

### Over SR-MPLS

Nothing extra — the base above advertises VPNv4 with an MPLS VPN label by default.

### Over SRv6 (delta only)

Add `encapsulation-type srv6` to the neighbor, and an SRv6 locator under the VRF:
```
router bgp 100
 neighbor 4.4.4.4
  address-family vpnv4 unicast
   encapsulation-type srv6           ! <-- SRv6
  !
 !
 vrf CUST-A
  address-family ipv4 unicast
   segment-routing srv6              ! <-- SRv6
    locator MAIN                     ! <-- SRv6
    alloc mode per-vrf               ! <-- SRv6
   !
  !
 !
!
```

### Verify
```
show bgp vpnv4 unicast summary           ! session Established, prefixes received
show route vrf CUST-A                     ! customer routes present
show segment-routing srv6 sid             ! (SRv6) uDT4 SID for the VRF
show cef vrf CUST-A <remote-ce> detail    ! (SRv6) SRv6 H.Encaps toward uDT4
! from CE: ping <remote-CE-loopback>
```

CE side: eBGP to the PE, advertising its own prefix(es). Same regardless of transport.

---

## Part 2 — L2VPN (EVPN-VPWS / E-Line, point-to-point)

### Common base (both transports)

```
interface GigabitEthernet0/0/0/2      ! CE-facing, now an L2 attachment circuit
 l2transport
!
router bgp 100
 address-family l2vpn evpn
 !
 neighbor 4.4.4.4
  address-family l2vpn evpn
  !
 !
!
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface GigabitEthernet0/0/0/2
   ! neighbor line differs per transport — see below
  !
 !
!
```
> New XR interfaces can come up admin-down — `no shutdown` the AC when pasting.

### Over SR-MPLS

Use the crossed AC-ID form (this PE `source 1 target 4`; the other PE swaps them):
```
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface GigabitEthernet0/0/0/2
   neighbor evpn evi 200 target 4 source 1
  !
 !
!
```

### Over SRv6 (delta only)

Add an SRv6 locator under `evpn`, and use the SRv6 service form on the xconnect
(`service N` matches on both PEs):
```
evpn
 segment-routing srv6                 ! <-- SRv6
  locator MAIN
 !
!
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface GigabitEthernet0/0/0/2
   neighbor evpn evi 200 service 1 segment-routing srv6   ! <-- SRv6
  !
 !
!
```

### Verify
```
show l2vpn xconnect [detail]              ! state UP; (SRv6) Encapsulation SRv6, H.Encaps.L2.Red
show bgp l2vpn evpn                        ! EVPN Route-Type 1 (Ethernet A-D)
show segment-routing srv6 sid             ! (SRv6) uDX2 SID for the service
! from CE: ping <remote-CE> (same subnet) ; show arp (far-end MAC = truly L2)
```

CE side: the AC port gets an IP in a **shared subnet** with the remote CE (e.g.
`172.16.0.1/24` and `172.16.0.2/24`) — no routing protocol; they're L2-adjacent.

---

## Multipoint note (E-LAN)

VPWS above is point-to-point. For multipoint L2 (E-LAN — a virtual switch across many
sites) you'd use **EVPN bridging**: a `bridge-domain` with an `evi` instead of an
xconnect, which adds MAC learning and BUM flooding. In SRv6 it uses `uDT2U` (unicast)
and `uDT2M` (BUM) SIDs instead of `uDX2`. Not built in this lab — listed for
completeness.

---

## The one takeaway

Across all four cells, the service definition barely changes — what changes is a
handful of lines that select the transport (`encapsulation-type srv6` + per-VRF/EVI
locator for SRv6; nothing for SR-MPLS). That is the whole point of Segment Routing:
**services are decoupled from transport**, so the same VPN runs over MPLS or IPv6
with a few lines of difference and zero change in the core.
