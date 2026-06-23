# SR-MPLS + SRv6 Lab — Cisco IOS XRv9000 / EVE-NG

![Platform](https://img.shields.io/badge/platform-Cisco%20IOS%20XRv9000-1BA0D7?logo=cisco&logoColor=white)
![IOS XR](https://img.shields.io/badge/IOS%20XR-24.3.1-005073)
![EVE-NG](https://img.shields.io/badge/EVE--NG-Community%20%7C%20Pro-orange)
![Dataplane](https://img.shields.io/badge/dataplane-SR--MPLS%20%2B%20SRv6-success)
![Phases](https://img.shields.io/badge/phases-7-blueviolet)

> **How to use this lab:**
> Follow the phases one by one. Paste the config, run the verify commands, make sure it works — *then* move to the next phase. You don't need to understand everything upfront. The concepts will click as you build.
>
> Want to read the theory first? → [`docs/CONCEPTS.md`](docs/CONCEPTS.md)

---

## TL;DR — quick start

For those who already know IOS XR and just want to build it:

1. Draw the topology in EVE-NG (6 nodes, links in the [table below](#topology--draw-this-in-eve-ng)).
2. Paste each device's full config from [`configs/`](configs/) — `R1.txt`–`R4.txt`, `CE1.txt`, `CE2.txt` are cumulative (all 7 phases).
3. Verify end to end:

```
! on R1 — transport reachable
ping 4.4.4.4 source Loopback0

! on CE1 — L3VPN works
ping vrf CUST-A 22.22.22.22 source 11.11.11.11    ! (run from PE) 
ping 22.22.22.22 source 11.11.11.11                ! (run from CE1)

! on CE1 — L2VPN (EVPN-VPWS) works
ping 172.16.0.2 source 172.16.0.1
```

All `!!!!!` → the whole stack (IS-IS → SR-MPLS → TI-LFA → SR-TE → L3VPN → SRv6 → EVPN-VPWS) is up. Want to learn instead of speed-run? Skip this and follow the phases below.

---

## What this lab builds

A real service-provider network using **Segment Routing** — the modern way to do MPLS. You will build it from scratch in 7 phases:

```
Phase 1 → IS-IS (routing foundation)
Phase 2 → SR-MPLS (labels without LDP — the main idea)
Phase 3 → TI-LFA (automatic fast reroute)
Phase 4 → SR-TE (traffic engineering — steer traffic anywhere)
Phase 5 → L3VPN (carry customer IP traffic)
Phase 6 → SRv6 (next-gen: IPv6-based transport)
Phase 7 → EVPN-VPWS (carry customer Layer 2 traffic)
```

Each phase adds one thing. If something breaks, you only need to look at what you just added.

---

## Before you start

**You need:**
- EVE-NG (Community or Pro) installed and running
- Cisco IOS XRv9000 image loaded into EVE-NG (tested on 24.3.1)
- 6 nodes in your topology: R1, R2, R3, R4, CE1, CE2

**Good to know (but not required yet):**
- Basic IOS XR CLI — how to enter config mode, `commit`, `show` commands
- What IS-IS is (a routing protocol, like OSPF)
- What MPLS is (labels on packets for fast forwarding)

---

## Topology — draw this in EVE-NG

```
                +--------+
     +----------+   R2   +----------+
     |          |  (P)   |          |
     |          +---+----+          |
     |              |               |
+----+----+         |          +----+----+
|   R1    +---------+----------+   R4    |
|  (PE)   |    via R2–R3       |  (PE)   |
+----+----+                    +----+----+
     |          +--------+         |
     +----------+   R3   +---------+
     |          |  (P)   |         |
     |          +--------+         |
     |                             |
+----+----+                   +----+----+
|   CE1   |                   |   CE2   |
+---------+                   +---------+
```

**Roles:**
- **PE** (Provider Edge) = R1 and R4. These connect to customers (CE1, CE2).
- **P** (Provider core) = R2 and R3. These just forward packets — no customer config.
- **CE** (Customer Edge) = CE1 and CE2. Simple routers that don't know about MPLS.

**Core links to create in EVE-NG** (these carry IS-IS / SR / SRv6):

| Link | Interface on left | Interface on right | Subnet |
|------|-------------------|--------------------|--------|
| R1 – R2 | R1 Gi0/0/0/1 | R2 Gi0/0/0/0 | 10.12.0.0/30 |
| R1 – R3 | R1 Gi0/0/0/3 | R3 Gi0/0/0/2 | 10.13.0.0/30 |
| R2 – R3 | R2 Gi0/0/0/1 | R3 Gi0/0/0/0 | 10.23.0.0/30 |
| R2 – R4 | R2 Gi0/0/0/3 | R4 Gi0/0/0/2 | 10.24.0.0/30 |
| R3 – R4 | R3 Gi0/0/0/4 | R4 Gi0/0/0/3 | 10.34.0.0/30 |

**Customer links** (added in Phases 5–7):

| Link | Service | Interface on PE | Interface on CE | Subnet |
|------|---------|-----------------|-----------------|--------|
| R1 – CE1 | L3VPN (Phase 5) | R1 Gi0/0/0/0 | CE1 Gi0/0/0/1 | 192.168.11.0/30 |
| R4 – CE2 | L3VPN (Phase 5) | R4 Gi0/0/0/1 | CE2 Gi0/0/0/0 | 192.168.44.0/30 |
| R1 – CE1 | L2VPN / EVPN-VPWS (Phase 7) | R1 Gi0/0/0/2 | CE1 Gi0/0/0/2 | 172.16.0.0/24 |
| R4 – CE2 | L2VPN / EVPN-VPWS (Phase 7) | R4 Gi0/0/0/0 | CE2 Gi0/0/0/1 | 172.16.0.0/24 |

> Interface numbers above match the shipped configs in [`configs/`](configs/) exactly — wire EVE-NG the same way or the pasted configs won't line up.

---

## IP Address Plan — use exactly these addresses

### Loopbacks (one per router — very important for SR)

| Router | Loopback0 | SR Label | SRv6 Locator |
|--------|-----------|----------|--------------|
| R1 | 1.1.1.1/32 | **16001** | fcbb:bb00:1::/48 |
| R2 | 2.2.2.2/32 | **16002** | fcbb:bb00:2::/48 |
| R3 | 3.3.3.3/32 | **16003** | fcbb:bb00:3::/48 |
| R4 | 4.4.4.4/32 | **16004** | fcbb:bb00:4::/48 |

> **Why loopbacks matter:** In SR, label 16001 means "reach R1". Label 16004 means "reach R4". Every router in the network agrees on this. This is the whole magic of SR-MPLS.

### Links between routers

| Link | Subnet | R1 side | R2/R3/R4 side |
|------|--------|---------|---------------|
| R1–R2 | 10.12.0.0/30 | .1 | .2 |
| R1–R3 | 10.13.0.0/30 | .1 | .2 |
| R2–R3 | 10.23.0.0/30 | .1 (R2) | .2 (R3) |
| R2–R4 | 10.24.0.0/30 | .1 (R2) | .2 (R4) |
| R3–R4 | 10.34.0.0/30 | .1 (R3) | .2 (R4) |

### Customer links (Phases 5–7)

| Link | Subnet | PE side | CE side |
|------|--------|---------|---------|
| R1–CE1 | 192.168.11.0/30 | .1 | .2 |
| R4–CE2 | 192.168.44.0/30 | .1 | .2 |

| CE Router | Loopback0 | BGP AS |
|-----------|-----------|--------|
| CE1 | 11.11.11.11/32 | 65001 |
| CE2 | 22.22.22.22/32 | 65002 |

---

## Phase 1 — IS-IS Baseline

**Goal:** All 4 routers (R1–R4) can ping each other's loopbacks. This is the foundation everything else builds on.

**Config files:** [`configs/R1.txt`](configs/R1.txt) through [`configs/R4.txt`](configs/R4.txt) — paste the IS-IS section for each router.

**What you're configuring:**
- An IS-IS process named `CORE` on each router
- A unique NET address (like a router ID for IS-IS)
- `metric-style wide` — required for SR later
- Loopback0 as passive (advertise but don't form adjacency on it)
- All links as `point-to-point`

**Paste this on R1 (example):**
```
router isis CORE
 is-type level-2-only
 net 49.0001.0000.0000.0001.00
 address-family ipv4 unicast
  metric-style wide
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/1
  point-to-point
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/3
  point-to-point
  address-family ipv4 unicast
  !
 !
!
commit
```
> Change NET last octet: R1=`0001`, R2=`0002`, R3=`0003`, R4=`0004`.

**✅ Verify — run these on R1:**
```
show isis neighbors
show route 4.4.4.4/32
ping 4.4.4.4 source Loopback0
```

**You should see:**
- `show isis neighbors` → R2 and R3 in **Up** state
- `ping 4.4.4.4` → !!!!!  (5 exclamation marks = success)

---

## Phase 2 — SR-MPLS (Prefix-SIDs)

**Goal:** Each router gets a globally unique label. R1 can reach R4 using label 16004 — no LDP needed.

**What you're adding:** `segment-routing mpls` in IS-IS, and a `prefix-sid index` on each Loopback0.

**Add to each router's IS-IS config:**
```
router isis CORE
 address-family ipv4 unicast
  segment-routing mpls
 !
 interface Loopback0
  address-family ipv4 unicast
   prefix-sid index 1    ← change to 2, 3, 4 for R2, R3, R4
  !
 !
!
commit
```

**✅ Verify on R1:**
```
show isis segment-routing label table
show mpls forwarding
ping 4.4.4.4 source Loopback0
```

**You should see:**
- Label table showing `16001=R1`, `16002=R2`, `16003=R3`, `16004=R4`
- Forwarding table showing how to reach each label
- Ping still working ✅

> **What just happened:** The IS-IS routing protocol itself now distributes labels. No separate LDP process needed. "16004 means reach R4" — every router in the network agrees on this automatically.

---

## Phase 3 — TI-LFA Fast Reroute

**Goal:** If a link fails, traffic reroutes in under 50ms — automatically, no manual config per-path.

**What you're adding:** Two lines per interface in IS-IS.

**Add under each IS-IS interface (on R1, R2, R3, R4):**
```
router isis CORE
 interface GigabitEthernet0/0/0/X     ← repeat for all transit interfaces
  address-family ipv4 unicast
   fast-reroute per-prefix
   fast-reroute per-prefix ti-lfa
  !
 !
!
commit
```

**✅ Verify on R1:**
```
show isis fast-reroute 4.4.4.4/32 detail
show isis ti-lfa frr-summary
```

**You should see:** Every prefix has a **repair path** listed — a backup route using a different label stack.

**Test it:** Shut the R1–R2 link (`shutdown` on Gi0/0/0/1). Traffic to R4 should immediately use R3 path. No ping drops.

---

## Phase 4 — SR-TE (Traffic Engineering)

**Goal:** Force traffic from R1 to R4 to take the path R1→R2→R3→R4 — even though the shortest path is R1→R2→R4 or R1→R3→R4. No changes needed on R2, R3, or R4.

**What you're adding on R1 only:** A segment list (ordered list of labels) and a policy that uses it.

```
segment-routing
 traffic-eng
  segment-list SCENIC-R1-TO-R4
   index 10 mpls label 16002
   index 20 mpls label 16003
   index 30 mpls label 16004
  !
  policy R1-TO-R4-SCENIC
   color 100 end-point ipv4 4.4.4.4
   autoroute
    include ipv4 4.4.4.4/32
   !
   candidate-paths
    preference 100
     explicit segment-list SCENIC-R1-TO-R4
    !
   !
  !
 !
!
commit
```

**✅ Verify on R1:**
```
show segment-routing traffic-eng policy
traceroute 4.4.4.4 source 1.1.1.1
```

**You should see:** Traceroute hops: R1 → R2 (10.12.0.2) → R3 (10.23.0.2) → R4

> **What just happened:** R1 put 3 labels on the packet (16002, 16003, 16004). Each P router just popped the top label and forwarded. R2 and R3 didn't need any config change. This is the power of SR-TE — all the intelligence is at the headend.

---

## Phase 5 — L3VPN over SR-MPLS

**Goal:** CE1 and CE2 can ping each other even though they're on different sides of the provider network.

**What you're adding:**
- VRF `CUST-A` on R1 and R4
- iBGP session between R1 and R4 (VPNv4 address family)
- eBGP to CE1 (from R1) and CE2 (from R4)

**On R1 and R4 (VRF + BGP):**
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
router bgp 100
 bgp router-id 1.1.1.1    ← use 4.4.4.4 on R4
 address-family vpnv4 unicast
 !
 neighbor 4.4.4.4          ← use 1.1.1.1 on R4
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
  neighbor 192.168.11.2    ← CE1 IP (use 192.168.44.2 on R4)
   remote-as 65001         ← use 65002 on R4 side
   address-family ipv4 unicast
    route-policy PASS in
    route-policy PASS out
   !
  !
 !
!
commit
```

> ⚠️ **IOS XR requires a route-policy on eBGP sessions** — without it, all prefixes are blocked even if the session is up. Add this before the BGP config:
> ```
> route-policy PASS
>   pass
> end-policy
> ```

**✅ Verify:**
```
show bgp vrf CUST-A summary
show route vrf CUST-A
ping vrf CUST-A 22.22.22.22 source 11.11.11.11
```

> **Bonus — RD vs RT demo:** the shipped `R1.txt` / `R4.txt` also define a second VRF **`CUST-B`** (`rd 100:2`, `rt 100:2`). It carries no CE, it's there to make the Route-Distinguisher vs Route-Target distinction concrete: same import/export mechanics, different values. Compare `show bgp vpnv4 unicast rd 100:1` against `rd 100:2` to see how the RD keeps otherwise-identical prefixes unique across VRFs. Skip it if you only care about the core path.

---

## Phase 6 — L3VPN over SRv6 (uDT4)

**Goal:** The same VPN now uses SRv6 (IPv6-based) transport instead of MPLS labels. One IPv6 address replaces both the transport label and the VPN label.

**What you're adding:** SRv6 locators on all routers, then tell BGP to use SRv6 encoding.

**On all routers (R1–R4) — add SRv6 locator:**
```
segment-routing
 srv6
  encapsulation
   source-address fcbb:bb00:1::1   ← use ::2, ::3, ::4 for R2/R3/R4
  !
  locators
   locator MAIN
    micro-segment behavior unode psp-usd
    prefix fcbb:bb00:1::/48        ← use bb00:2, bb00:3, bb00:4
   !
  !
 !
!
router isis CORE
 address-family ipv6 unicast
  metric-style wide
  segment-routing srv6
   locator MAIN
  !
 !
 interface Loopback0
  address-family ipv6 unicast
  !
 !
 interface GigabitEthernet0/0/0/X  ← all transit interfaces
  address-family ipv6 unicast
  !
 !
!
commit
```

**On R1 and R4 — update BGP to use SRv6:**
```
router bgp 100
 neighbor 4.4.4.4        ← 1.1.1.1 on R4
  address-family vpnv4 unicast
   encapsulation-type srv6
  !
 !
 vrf CUST-A
  address-family ipv4 unicast
   segment-routing srv6
    locator MAIN
    alloc mode per-vrf
   !
  !
 !
!
commit
```

**✅ Verify:**
```
show segment-routing srv6 sid
show bgp vrf CUST-A 11.11.11.11/32 detail
ping vrf CUST-A 22.22.22.22 source 11.11.11.11
```

---

## Phase 7 — EVPN-VPWS L2VPN over SRv6 (uDX2)

**Goal:** CE1 and CE2 connect as if they're on the same Ethernet cable — Layer 2 transparent service. No IP config needed on CE routers for this service.

> **Heads-up on interfaces:** the L2 attachment circuit is on a *different* port on each PE — **R1 uses Gi0/0/0/2**, **R4 uses Gi0/0/0/0** (R4's Gi0/0/0/1 is already the L3VPN port). Match the [topology table](#topology--draw-this-in-eve-ng).

**What you're adding on R1 and R4:**
```
evpn
 segment-routing srv6
  locator MAIN
 !
!
l2vpn
 xconnect group EVPN-VPWS
  p2p CE1-CE2-L2
   interface GigabitEthernet0/0/0/2    ← Gi0/0/0/0 on R4
   neighbor evpn evi 200 service 1 segment-routing srv6
  !
 !
!
commit
```

**Set the attachment-circuit port as L2:**
```
interface GigabitEthernet0/0/0/2    ← Gi0/0/0/0 on R4
 l2transport
!
commit
```

**On the CE side — assign an IP in the shared 172.16.0.0/24 L2 segment:**
```
! CE1 — on Gi0/0/0/2
interface GigabitEthernet0/0/0/2
 ipv4 address 172.16.0.1 255.255.255.0
!
! CE2 — on Gi0/0/0/1
interface GigabitEthernet0/0/0/1
 ipv4 address 172.16.0.2 255.255.255.0
!
commit
```

**✅ Verify:**
```
show evpn evi detail
show l2vpn xconnect detail
ping 172.16.0.2 source 172.16.0.1    ← run on CE1
```

---

## All Phases Complete ✅

If all pings work, you have built a full service-provider network from scratch:

```
✅ Phase 1 — IS-IS baseline
✅ Phase 2 — SR-MPLS (global labels, no LDP)
✅ Phase 3 — TI-LFA (auto fast reroute <50ms)
✅ Phase 4 — SR-TE (traffic steering at headend)
✅ Phase 5 — L3VPN over SR-MPLS
✅ Phase 6 — L3VPN over SRv6
✅ Phase 7 — EVPN-VPWS L2VPN over SRv6
```

**Now go back and read the theory:**
- [`docs/CONCEPTS.md`](docs/CONCEPTS.md) — why each phase works the way it does
- [`docs/PARAMETERS.md`](docs/PARAMETERS.md) — what each config knob does and what breaks if it's wrong
- [`docs/SR-MPLS-vs-SRv6.md`](docs/SR-MPLS-vs-SRv6.md) — deep comparison of the two data planes

---

## Troubleshooting Quick Reference

| Problem | First command to run |
|---------|----------------------|
| IS-IS neighbor not up | `show isis neighbors` → check interface/area |
| Loopback not in routing table | `show isis database` → check prefix-sid advertised |
| SR label wrong | `show isis segment-routing label table` |
| BGP neighbor not up | `show bgp neighbors x.x.x.x` → check AS, update-source |
| VPN prefix missing | `show bgp vrf CUST-A summary` → check route-target |
| eBGP up but no prefixes | Add `route-policy PASS` — IOS XR requires this |
| SRv6 SID not allocated | `show segment-routing srv6 sid` → check locator config |
| EVPN PW down | `show evpn evi detail` → check EVI number matches both PEs |

---

## Repository Structure

```
sr-mpls-iosxr-eveng-lab/
├── README.md            ← start here (lab guide)
├── configs/
│   ├── R1.txt           ← full cumulative config (all 7 phases)
│   ├── R2.txt
│   ├── R3.txt
│   ├── R4.txt
│   ├── CE1.txt
│   └── CE2.txt
├── docs/
│   ├── CONCEPTS.md      ← read after the lab — theory explained
│   ├── PARAMETERS.md    ← every knob and what breaks if wrong
│   ├── SR-MPLS-vs-SRv6.md
│   ├── PE-Template.md
│   └── VPN-Config-Guide.md
└── notes/
    └── README.md        ← session log
```

---

## Related Lab

**[MPLS-TE Lab (RSVP-TE)](https://github.com/bosamart/Cisco-IOS-XR-MPLS-Traffic-Engineering-Lab)** — same topology, same VPN, but using classic LDP + RSVP-TE. Build both to understand why the industry moved to Segment Routing.

---

*Built by [@bosamart](https://github.com/bosamart) — IP Network Engineer, Cambodia*