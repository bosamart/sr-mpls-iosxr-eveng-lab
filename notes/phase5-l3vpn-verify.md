# Phase 5 — L3VPN (MP-BGP VPNv4) over SR-MPLS: Verification Log

**Date:** 2026-06-24
**Objective:** Carry a real customer VRF (`CUST-A`) across the SR core so CE1
(11.11.11.11, AS 65001) and CE2 (22.22.22.22, AS 65002) reach each other, using an
iBGP VPNv4 session between the R1 and R4 loopbacks and eBGP to each CE.

**Result:** ✅ **PASS** — VPNv4 session up (2 prefixes received), both customer
loopbacks in `vrf CUST-A`, the remote prefix resolves `nexthop in vrf default`, and
**CE1→CE2 ping succeeds 5/5**. The CUST-B VRF demonstrates the RD making identical
prefixes unique.

---

## Commands run

```
show bgp vpnv4 unicast summary     ! VPNv4 session + prefix count
show bgp vpnv4 unicast             ! the VPN prefixes per RD
show route vrf CUST-A              ! customer RIB on R1
ping 22.22.22.22 source 11.11.11.11   ! from CE1 — end to end
```

---

## VPNv4 session (R1)

```
RP/0/RP0/CPU0:R1#show bgp vpnv4 unicast summary
Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
4.4.4.4           0   100      15      14       18    0    0 00:09:04          2
```

## VPN prefixes — note the same prefix under two RDs (R1)

```
RP/0/RP0/CPU0:R1#show bgp vpnv4 unicast
Route Distinguisher: 100:1 (default for vrf CUST-A)
*> 11.11.11.11/32     192.168.11.2             0             0 65001 i
*>i22.22.22.22/32     4.4.4.4                  0    100      0 65002 i
*> 192.168.11.0/30    0.0.0.0                  0         32768 ?
*>i192.168.44.0/30    4.4.4.4                  0    100      0 ?
Route Distinguisher: 100:2 (default for vrf CUST-B)
*> 192.168.11.0/30    0.0.0.0                  0         32768 ?
Processed 5 prefixes, 5 paths
```

## Customer RIB (R1)

```
RP/0/RP0/CPU0:R1#show route vrf CUST-A
B    11.11.11.11/32 [20/0] via 192.168.11.2, 00:08:27
B    22.22.22.22/32 [200/0] via 4.4.4.4 (nexthop in vrf default), 00:08:11
C    192.168.11.0/30 is directly connected, GigabitEthernet0/0/0/0
B    192.168.44.0/30 [200/0] via 4.4.4.4 (nexthop in vrf default), 00:08:12
```

## End-to-end — CE1 to CE2 (CE1)

```
RP/0/RP0/CPU0:CE1#ping 22.22.22.22 source 11.11.11.11
!!!!!  Success rate is 100 percent (5/5), round-trip min/avg/max = 8/29/106 ms
```

---

## Analysis

- **VPNv4 session up, routes exchanged.** `St/PfxRcd = 2` — R1 received CE2's loopback
  and the R4–CE2 subnet from R4 over the loopback-to-loopback iBGP session. ✔
- **RD-vs-RT demo, made concrete.** `192.168.11.0/30` appears under **both** RD `100:1`
  (CUST-A) **and** RD `100:2` (CUST-B). Same IPv4 prefix, two VRFs — the **RD** is what
  keeps them unique inside MP-BGP. (RT, separately, controls which VRFs import the
  route.) This is the whole point of the second VRF. ✔
- **The key line — `nexthop in vrf default`.** R1's route to `22.22.22.22/32` resolves
  its BGP next-hop **4.4.4.4 in the global table**, not in the VRF. That's the
  two-label model: an outer SR transport label reaches R4, an inner VPN label selects
  the VRF. The core forwards on the transport SID only and never sees customer routes
  — why SR/MPLS L3VPN scales. ✔
- **PE–CE eBGP works.** `11.11.11.11/32` learned `via 192.168.11.2` (CE1), AS-path
  `65001` — eBGP up and passing prefixes (the IOS XR `route-policy PASS` in/out is in
  place). ✔
- **End-to-end proven.** CE1 pinging CE2's loopback succeeds 5/5 across the SR core —
  the real customer-service test, not just control-plane state. ✔

**Conclusion:** L3VPN over SR-MPLS verified end to end. Proceed to Phase 6 — carry the
*same* VRF over SRv6 instead of MPLS labels.
