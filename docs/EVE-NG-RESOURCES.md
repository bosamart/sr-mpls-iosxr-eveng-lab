# EVE-NG Resources — sizing this lab on a constrained host

Real-world notes for running the lab (and the Phase 8 extension) without running out
of RAM. Tested mindset: **XRv9000 is RAM-hungry, and RAM cannot be oversubscribed** —
if it swaps, XR processes get OOM-killed and nodes crash. CPU you *can* oversubscribe;
RAM you cannot.

## Host / VM budget (example: 64 GB Proxmox host)

| Layer | RAM | Notes |
|---|---|---|
| Proxmox host + (ZFS ARC) | ~8 GB | If on **ZFS**, cap the ARC (`zfs_arc_max` ≈ 4 GB) or it silently eats host RAM. |
| EVE-NG VM | **56 GB** | Leave the host ~8 GB. |
| EVE-NG Ubuntu OS | ~3–4 GB | Overhead inside the VM, before any nodes. |
| QEMU per-node overhead | ~0.2–0.5 GB each | Real usage is a bit above the sum of guest RAM. |
| **Usable for nodes** | **~50 GB** | Plan node RAM to fit *under* this. |

## Per-node RAM

| Node | Role | Image | RAM | vCPU |
|---|---|---|---|---|
| R1, R4 | PE | XRv9000 | 8 GB | 2 |
| R2, R3 | P (core) | XRv9000 | 8 GB | 2 |
| PCE | SR-PCE (Phase 8) | XRv9000 | 8 GB | 2 |
| CE1, CE2 | customer (L2+L3) | XRv9000 | 8 GB | 2 |
| CE3, CE4 | customer (L3 only, light) | **CSR 1000v / IOS-XE** | **4 GB** | 1 |

Set RAM/vCPU per node in EVE-NG: *Edit node → RAM / CPU*, then reboot the node. The two
PEs also need their interface count raised to **5** (to expose `Gi0/0/0/4`) — *Edit node →
Ethernets = 5*, done once while the node is stopped.

## Two CE sets, both cabled, power only what you need

CE3/CE4 are **not** replacements — they're **additional L3VPN sites on their own PE ports
and IPs**, so nothing is ever re-cabled. You leave everything wired and just power on the
CEs a given test needs (cabling costs no RAM; only powered-on nodes do).

| CE set | Image | RAM | Services | Use it for |
|---|---|---|---|---|
| **CE1 / CE2** | XRv9000 | 8 GB ea | L3VPN **+ EVPN-VPWS (L2)** | Phase 7 (L2VPN), XR-specific behavior |
| **CE3 / CE4** | CSR 1000v | 4 GB ea | L3VPN only | Phases 5–6 (L3VPN) and **Phase 8 (ODN)** |

New wiring for CE3/CE4 (everything else unchanged):

| Link | PE port (new) | CSR port | Subnet | CE Lo0 / AS / VRF |
|---|---|---|---|---|
| R1–CE3 | R1 `Gi0/0/0/4` (.1) | CE3 `Gi1` (.2) | 192.168.30.0/30 | 33.33.33.33 · AS 65003 · CUST-A |
| R4–CE4 | R4 `Gi0/0/0/4` (.1) | CE4 `Gi1` (.2) | 192.168.40.0/30 | 44.44.44.44 · AS 65004 · CUST-A |

> Because CE3/CE4 are in **CUST-A**, they reach CE1/CE2 and each other over the SR core.
> For **Phase 8**, CE4 always advertises `44.44.44.44`; R4 colors it → reliably triggers
> ODN on R1 (no flapping XR CE needed). **Phase 7 (EVPN-VPWS) still needs CE1/CE2** — the
> CSR CEs are L3-only.

**Running budgets (power on only what the test needs):**

| Active set | Sum | Fits 56 GB VM? |
|---|---|---|
| 4 core + CE3/CE4 (L3 lab, no PCE) | 32 + 8 = **40 GB** | ✅ roomy |
| 4 core + PCE + CE3/CE4 (**Phase 8**) | 40 + 8 = **48 GB** | ✅ comfortable |
| 4 core + CE1/CE2 (**Phase 7 L2VPN**, no PCE) | 32 + 16 = **48 GB** | ✅ comfortable |
| 4 core + PCE + CE1/CE2 | 40 + 16 = **56 GB** | ⚠️ tight — use KSM |
| everything powered at once | 64+ GB | ❌ over budget — don't |

> Tip: the **PCE** (8 GB) is only needed for **Phase 8**; leave it off otherwise.
> The **XR CEs** (CE1/CE2) are only needed for **Phase 7**; for everything else run the
> light CSR pair. You almost never need PCE *and* the XR CEs at the same time.

## Enable KSM (free RAM back)

Your core nodes boot from the **same** XRv9000 image, so large chunks of their memory are
identical. **Kernel Same-page Merging** dedupes those pages and reclaims several GB.

On the EVE-NG Ubuntu host:
```
# quick on:
echo 1 | sudo tee /sys/kernel/mm/ksm/run
# or persistent + tuned:
sudo apt-get install -y ksmtuned
sudo systemctl enable --now ksmtuned
```
Give it a few minutes after all nodes are up, then check reclaim with `free -h` and
`cat /sys/kernel/mm/ksm/pages_sharing`.

## Quick gotchas

- **Don't let it swap.** If `free -h` shows swap in use while nodes run, you're over —
  drop a node (the PCE, or use the CSR CE pair) rather than swap.
- **CPU:** 2 vCPU per XRv9000 is plenty for a control-plane lab; CPU can oversubscribe.
- **CSR licensing:** runs throughput-limited without a license — irrelevant for a
  control-plane lab; eBGP and the L2 segment work fine.
- **IOS-XE vs IOS-XR:** the CSR CEs use `GigabitEthernet1/2`, need `feature`-free classic
  IOS BGP, and have **no** `route-policy PASS` requirement (that XR gotcha is gone).

---

*Companion to [`../README.md`](../README.md) and [`PHASE8-SR-PCE-ODN.md`](PHASE8-SR-PCE-ODN.md).
CSR drop-in configs: [`../configs/CE3-csr.txt`](../configs/CE3-csr.txt),
[`../configs/CE4-csr.txt`](../configs/CE4-csr.txt).*
