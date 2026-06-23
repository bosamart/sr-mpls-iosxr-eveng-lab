# VPN Configuration Guide — L2 & L3 VPN over SR-MPLS and SRv6

Quick-reference for all four service/transport combinations. Examples use lab values
(R1 `1.1.1.1` / R4 `4.4.4.4`, AS 100, locator `MAIN`). Configs show one PE — the other mirrors.

| Service | Transport | Service SID | Delta vs SR-MPLS |
|---|---|---|---|
| L3VPN (VPNv4) | SR-MPLS | MPLS VPN label | — (baseline) |
| L3VPN (VPNv4) | SRv6 | `uDT4` | `encapsulation-type srv6` + per-VRF locator |
| L2VPN VPWS | SR-MPLS | PW label | `l2vpn evpn` AF + xconnect `target/source` |
| L2VPN VPWS | SRv6 | `uDX2` | `evpn` SRv6 locator + xconnect `service … srv6` |

---

## Part 1 — L3VPN (MP-BGP VPNv4)

### Common base (both transports)

```
vrf CUST-A
 address-family ipv4 unicast
  import route-target 100:1
  export route-target 100:1
!
route-policy PASS
  pass
end-policy
!
router bgp 100
 address-family vpnv4 unicast
 neighbor 4.4.4.4
  remote-as 100
  update-source Loopback0
  address-family vpnv4 unicast
 vrf CUST-A
  rd 100:1
  address-family ipv4 unicast
   redistribute connected
  neighbor 192.168.11.2
   remote-as 65001
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
```

### Over SR-MPLS — nothing extra. VPNv4 uses MPLS VPN label by default.

### Over SRv6 — delta only

```
router bgp 100
 neighbor 4.4.4.4
  address-family vpnv4 unicast
   encapsulation-type srv6           ! <-- SRv6
 vrf CUST-A
  address-family ipv4 unicast
   segment-routing srv6              ! <-- SRv6
    locator MAIN
    alloc mode per-vrf
```

### Verify

```
show bgp vpnv4 unicast summary
show route vrf CUST-A
show segment-routing srv6 sid        ! (SRv6) see uDT4 SID
show cef vrf CUST-A <prefix> detail  ! (SRv6) H.Encaps toward uDT4
! from CE: ping <remote-CE-loopback>
```

---

## Part 2 — L2VPN (EVPN-VPWS / E-Line)

### Common base (both transports)

```
interface GigabitEthernet0/0/0/2
 l2transport
!
router bgp 100
 address-family l2vpn evpn
 neighbor 4.4.4.4
  address-family l2vpn evpn
```

### Over SR-MPLS

```
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface GigabitEthernet0/0/0/2
   neighbor evpn evi 200 target 4 source 1   ! R4 mirrors: target 1 source 4
```

### Over SRv6 — delta only

```
evpn
 segment-routing srv6                ! <-- SRv6
  locator MAIN
!
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface GigabitEthernet0/0/0/2
   neighbor evpn evi 200 service 1 segment-routing srv6   ! <-- SRv6
```

### Verify

```
show l2vpn xconnect detail
show segment-routing srv6 sid        ! (SRv6) see uDX2 SID
! from CE: ping <remote-CE> (same subnet)
! from CE: show arp  (far-end MAC = truly L2)
```

---

## Key takeaway

Across all four combinations, the **service config barely changes** — a handful of
lines select the transport. Services are decoupled from transport.
