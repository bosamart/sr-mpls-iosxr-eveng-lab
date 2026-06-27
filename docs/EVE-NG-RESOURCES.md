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
| CE1, CE2 | customer | XRv9000 | 8 GB | 2 |
| CE3, CE4 | customer (light) | **CSR 1000v / IOS-XE** | **4 GB** | 1 |

**Two CE pairs, run one at a time** (see below). Set RAM/vCPU per node in EVE-NG:
*Edit node → RAM / CPU*, then reboot the node.

## The CE-pair power strategy (the key trick)

The PEs only have ports for **one** CE pair, and running all four CEs would blow the RAM
budget. So the lab defines two interchangeable customer pairs:

- **CE1 / CE2 — XRv9000 (8 GB each).** The canonical configs in the repo
  (`configs/CE1.txt`, `CE2.txt`). Power these on when you want the exact XR CE behavior.
- **CE3 / CE4 — CSR 1000v (4 GB each).** Light drop-in replacements
  (`configs/CE3-csr.txt`, `CE4-csr.txt`) with **identical addressing**, cabled to the
  **same PE ports**. The PEs are unchanged and never know the difference.

> **Rule: run CE3/CE4 _or_ CE1/CE2 — never both.** They share the same PE ports
> (R1 Gi0/0/0/0 + Gi0/0/0/2, R4 Gi0/0/0/1 + Gi0/0/0/0) and the same IPs, so only one
> pair can be cabled/powered at a time. In EVE-NG, keep the off pair simply **not started**.

**Running budgets:**

| Active set | Sum | Fits 56 GB VM? |
|---|---|---|
| 5 core + CE3/CE4 (CSR) | 40 + 8 = **48 GB** | ✅ comfortable |
| 5 core + CE1/CE2 (XR) | 40 + 16 = **56 GB** | ⚠️ tight — use KSM, or skip the PCE when not on Phase 8 |
| all 9 nodes at once | **64 GB** | ❌ over budget — don't |

> Tip: the **PCE** (8 GB) is only needed for **Phase 8**. For Phases 1–7 leave it off and
> you reclaim 8 GB — which makes the XR CE pair (CE1/CE2) fit easily.

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
