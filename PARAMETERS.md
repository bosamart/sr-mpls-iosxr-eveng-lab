# Parameter Reference — Understanding Every Knob

The antidote to "I followed the template but don't know why." Every parameter below is
tagged with its **scope** — the single most useful thing to know about any config
value:

- **[DOMAIN-WIDE]** — must be identical on every node in the network
- **[PER-DEVICE]** — unique to each router
- **[PER-SERVICE]** — unique to each VPN/service
- **[LOCAL]** — meaningful only on that box; doesn't need to match anyone

Sort a parameter into the right bucket and you understand what it *is*.

---

## 1. L3VPN parameters

### Route Distinguisher — `rd 100:1`  **[PER-SERVICE]** (per-VRF, per-PE)
**Job:** make customer prefixes globally unique in BGP. Two customers can both use
`192.168.1.0/24`; the RD is prepended so they become distinct VPNv4 routes
(`100:1:192.168.1.0` vs `100:2:192.168.1.0`).
**Key point:** the RD does **nothing** for VPN membership — it *only* guarantees
uniqueness. People confuse this constantly.
**If wrong:** two customers sharing an RD with overlapping IPs collide.
**Must-match?** No — it can differ per PE (some designs deliberately use a unique RD
per PE). Keeping it consistent is just for clarity.

### Route Targets — `import/export route-target 100:1`  **[PER-SERVICE]**
**Job:** this is the *actual* control for VPN membership. A PE *exports* its VRF
routes stamped with the export-RT; a remote PE *imports* any route whose RT matches
its import-RT.
**The RD-vs-RT rule:** RD = *uniqueness*; RT = *membership*. Different jobs entirely.
**If wrong:** export-RT on one PE not in the other's import list → routes are sent but
never imported → VPN looks "up" but has no connectivity.
**Must-match?** Yes — one PE's export-RT must be in the other PE's import-RT. For a
simple any-to-any VPN, import = export = same value everywhere. (Hub-and-spoke uses
deliberately asymmetric RTs.)

### `route-policy PASS … pass` applied `in`/`out`  **[LOCAL]** (IOS XR eBGP only)
**Job:** IOS XR has **no implicit permit** on eBGP — by default it denies all routes
in and out. A permit-all policy must be applied to allow anything.
**If wrong (missing):** the eBGP session comes up but exchanges **zero** prefixes —
the classic XR "BGP up, no routes" trap.
**Must-match?** No, it's local to each PE. Note: iBGP does *not* need this; only eBGP.
This is XR-specific (IOS/IOS-XE has an implicit permit).

---

## 2. SR-MPLS parameters

### SRGB (Segment Routing Global Block) — default `16000–23999`  **[DOMAIN-WIDE]**
**Job:** the label range reserved for prefix-SIDs. A node's label = SRGB base + its
index, so index 4 → label 16004.
**If wrong:** different SRGBs on different nodes break the "globally agreed label"
property — 16004 would no longer mean R4 everywhere.
**Must-match?** Yes, domain-wide. (XR uses the default automatically, so it's usually
consistent without you touching it.)

### `prefix-sid index 4` (on Loopback0)  **[PER-DEVICE]**
**Job:** the offset into the SRGB that identifies this node. Index 4 → label 16004.
**Why an index, not the absolute label?** So you can change the SRGB base later
without renumbering every node.
**If wrong:** duplicate index = two nodes claim the same label = broken forwarding.
**Must-match?** Must be **unique** per node.

### NET address — `net 49.0001.0000.0000.0004.00`  **[PER-DEVICE]**
This is an OSI/CLNS address, because IS-IS is CLNS-based, not IP-based. Read it in
parts:
- `49` = AFI — "private / locally administered" (the OSI equivalent of RFC1918).
- `0001` = the **area** ID.
- `0000.0000.0004` = the **system ID** (6 bytes) — must be unique per node. Here it
  encodes the node number (…0004 = R4). A common convention is to derive it from the
  loopback.
- `00` = NSEL — always `00` for a router.
**If wrong:** duplicate system ID breaks the IS-IS database/adjacencies.
**Must-match?** System ID unique per node. Area must match for L1 adjacencies (in
this L2-only lab the area is the same anyway).

---

## 3. SRv6 parameters

### Locator — `prefix fcbb:bb00:4::/48`  **[PER-DEVICE]** (with a **[DOMAIN-WIDE]** block)
**Job:** the IPv6 prefix block a node carves all its SIDs from. IS-IS advertises it
like any IPv6 route, so others route toward it.
**Structure (uSID, f3216):** `fcbb:bb00` = the 32-bit uSID **block** (same on every
node), and the next 16 bits (`…:4:`) = the **node ID** (unique). So the block is
shared, the node part is unique.
**If wrong:** duplicate locator = two nodes claim the same SID space.
**Must-match?** Block domain-wide; node portion unique.

### The SIDs: `uN` / `uA` / `uDT4` / `uDX2`  **[AUTO-DERIVED]** — you don't type these
They are allocated automatically from the locator; you only configure the locator and
*which services use it*. The router builds the SIDs:
- **uN** — node SID (the locator itself, `fcbb:bb00:4::`). "Transit to R4." The SRv6
  twin of the prefix-SID. *(allocated by SID manager)*
- **uA** — adjacency SID (`fcbb:bb00:4:e000::`), one per link. "Use this exact link."
  The SRv6 twin of the adjacency-SID. *(allocated by IS-IS)*
- **uDT4** — L3VPN service SID (`…:e002::`). "Decap + IPv4 VRF lookup." *(allocated by
  BGP when you bind the VRF)*
- **uDX2** — L2VPN/VPWS service SID (`…:e003::`). "Decap + hand frame to this L2
  port." *(allocated by l2vpn when you bind the xconnect)*
The `e000/e001/e002/e003` are function IDs the router assigns inside the locator.
**The thing to understand:** SIDs are an *output*, not an input — you configure the
locator and the service bindings; the SIDs appear on their own.

### `alloc mode per-vrf`  **[PER-SERVICE]**
**Job:** how many service SIDs to allocate for the VRF.
- `per-vrf` — one `uDT4` for the whole VRF; the egress PE does a route lookup in the
  VRF table. Fewer SIDs. (This is what the lab uses — the simplest, standard choice.)
- `per-ce` — a separate SID per CE/next-hop; the egress can forward without a lookup.
  More SIDs, slightly faster.
**Analogy:** this is exactly the MPLS "per-VRF label" vs "per-prefix label" choice.
**If wrong:** nothing breaks in a simple lab — `per-vrf` is the safe default.

---

## 4. EVPN (L2VPN) parameters

### `evi 200` (EVPN Instance)  **[PER-SERVICE]**
**Job:** identifies the EVPN instance — a VPN-ID for the L2 service. It's the L2 twin
of the VRF/RD pairing for L3.
**If wrong:** different evi on each PE → the endpoints don't join the same service.
**Must-match?** Same evi identifies the service across PEs.

### `target 4 source 1` (SR-MPLS VPWS form)  **[PER-ENDPOINT]** — must **cross**
**Job:** identify the two attachment circuits of the point-to-point pseudowire.
`source` = local AC-ID, `target` = remote AC-ID. Read it as "I am AC 1, connecting to
AC 4."
**The crossing rule:** R1 `source 1 target 4` must mirror R4 `source 4 target 1`. The
local source on one side equals the target on the other.
**If wrong:** they don't cross-match → pseudowire stays down.

### `service 1 segment-routing srv6` (SRv6 VPWS form)  **[PER-SERVICE]** — must **match**
**Job:** in the SRv6 form, a single symmetric `service` ID identifies the pseudowire.
**Difference from MPLS:** SRv6 uses one matching service ID on *both* ends, not the
crossed source/target pair. Both PEs use `service 1`.
**If wrong:** mismatched service IDs → pseudowire down.

---

## One-page summary

| Parameter | Scope | Must match? | Breaks if wrong |
|---|---|---|---|
| `rd` | per-service / per-PE | no | overlapping customer IPs collide |
| route-target import/export | per-service | yes (export↔import) | VPN up but no connectivity |
| route-policy PASS | local (XR eBGP) | no | eBGP up, zero prefixes |
| SRGB | domain-wide | yes | "16004 = R4" no longer holds |
| prefix-sid index | per-device | unique | duplicate label, broken forwarding |
| NET / system-id | per-device | unique | IS-IS database/adjacency errors |
| SRv6 locator | per-device (shared block) | block matches, node unique | duplicate SID space |
| uN/uA/uDT4/uDX2 | auto-derived | n/a | (not configured directly) |
| alloc mode | per-service | no | safe default = per-vrf |
| evi | per-service | yes | endpoints don't join service |
| source/target (MPLS VPWS) | per-endpoint | must cross | PW down |
| service (SRv6 VPWS) | per-service | must match | PW down |
