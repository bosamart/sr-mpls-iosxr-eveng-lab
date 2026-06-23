# Parameter Reference — Understanding Every Knob

Every parameter tagged with its **scope**:
- **[DOMAIN-WIDE]** — must be identical on every node
- **[PER-DEVICE]** — unique to each router
- **[PER-SERVICE]** — unique to each VPN/service
- **[LOCAL]** — meaningful only on that box

---

## L3VPN parameters

### `rd 100:1` — [PER-SERVICE]
Makes customer prefixes globally unique in BGP. Two customers can use the same prefix; the RD distinguishes them. **Does NOT control VPN membership** — only uniqueness.

### `import/export route-target 100:1` — [PER-SERVICE]
The *actual* control for VPN membership. Export-RT on one PE must be in import-RT list on the other. **RD = uniqueness; RT = membership.**

### `route-policy PASS … pass` — [LOCAL] (IOS XR eBGP only)
IOS XR has no implicit permit on eBGP — denies all by default. Missing this = BGP session up but zero prefixes exchanged. Only needed for eBGP, not iBGP.

---

## SR-MPLS parameters

### SRGB (default 16000–23999) — [DOMAIN-WIDE]
Label range for prefix-SIDs. Label = SRGB base + index. Different SRGBs across nodes breaks the "globally agreed label" property.

### `prefix-sid index N` — [PER-DEVICE] (must be unique)
Offset into the SRGB for this node. Index 4 → label 16004. Duplicate index = two nodes claim the same label = broken forwarding.

### NET address — [PER-DEVICE] (system-id must be unique)
`49` = AFI, `0001` = area, `0000.0000.000X` = system-id (unique per node), `00` = NSEL. Duplicate system-id breaks IS-IS.

---

## SRv6 parameters

### Locator `fcbb:bb00:X::/48` — [PER-DEVICE] (shared block, unique node portion)
IPv6 prefix a node carves SIDs from. `fcbb:bb00` = domain block (same everywhere), last 16 bits = node ID (unique). Advertised by IS-IS like a route.

### uN / uA / uDT4 / uDX2 — [AUTO-DERIVED]
Allocated automatically from the locator — you configure the locator and service bindings; the SIDs appear on their own.

### `alloc mode per-vrf` — [PER-SERVICE]
One `uDT4` per VRF (safe default). Per-CE allocates a SID per next-hop (more SIDs, slightly faster egress).

---

## EVPN parameters

### `evi 200` — [PER-SERVICE] (must match on both PEs)
Identifies the EVPN instance (the L2 VPN-ID). Different evi = endpoints don't join the same service.

### SR-MPLS VPWS: `target 4 source 1` — must **cross**
R1 `source 1 target 4` ↔ R4 `source 4 target 1`. Local source = remote's target.

### SRv6 VPWS: `service 1` — must **match** on both PEs
Single symmetric service ID. Both PEs use `service 1`. (Unlike MPLS form, no crossing.)

---

## One-page summary

| Parameter | Scope | Must match? | Breaks if wrong |
|---|---|---|---|
| `rd` | per-service / per-PE | no | overlapping customer IPs collide |
| route-target import/export | per-service | yes (export↔import) | VPN up but no connectivity |
| route-policy PASS | local (XR eBGP) | no | eBGP up, zero prefixes |
| SRGB | domain-wide | yes | "16004 = R4" breaks |
| prefix-sid index | per-device | unique | duplicate label, broken forwarding |
| NET / system-id | per-device | unique | IS-IS errors |
| SRv6 locator | per-device (shared block) | block matches, node unique | duplicate SID space |
| evi | per-service | yes | endpoints don't join service |
| source/target (MPLS VPWS) | per-endpoint | must cross | PW down |
| service (SRv6 VPWS) | per-service | must match | PW down |
