# Capability status board

> [!WARNING]
> **Status as of the corpus freeze, 2026-07-28**
>
> Everything below describes the state of things at the moment the sources were captured: the
> chat archive ends 2026-07-28 at 00:01 and the repository snapshot was taken shortly after
> midnight UTC the same day. **Nothing on this page has been re-verified since.** Drift has
> already been observed: the remote `ecc` branch was force-updated within a day of the snapshot,
> so the description of it below is a snapshot description, not a live one. See
> [methodology](../appendix/methodology.md).

**What this page covers.** One table for every capability anyone has tried to unlock on the
NVIDIA CMP 170HX, with its current status, the mechanism that achieves it, and where to read the
detail. If you link one page from this wiki, link this one.

The short version: **compute and memory are solved and shipping.** An 8 GB card
(`10de:20c2`) becomes a 64 GB card and a 10 GB card (`10de:2082`) becomes a 40 GB card, with full
SM throughput restored, using a patched build of the NVIDIA open kernel modules. **PCIe link speed
(Gen1 to Gen2) is solved and shipping**, merged to `master` on 2026-07-29. **PCIe link width
(x4 to x16) is solved but requires soldering 24 capacitors**, and is a completely separate
achievement from link speed. **Everything else** (80 GB on a 10 GB card, Gen3, Gen4, NVLink, ECC,
P2P) is either unstable, blocked by a fuse with no known lever, unresolved, or untried. The 80 GB
case is the subtle one: a script-driven coherent register set does reach real memory past 40 GiB,
but nothing installable does, and 40 GB remains the supported configuration for a 10 GB card.

## How to read the status column

| Label | Meaning |
|---|---|
| **Working (shipped)** | In cmpunlocker `master`. Reproduced by many independent testers. |
| **Working (hardware)** | Requires a physical board modification. No software component. |
| **Working (shipped, partially)** | In `master`, but only part of the capability is delivered. |
| **Working (with caveats)** | Works, with a documented limitation you must plan around. |
| **Working (hardware + shipped)** | Needs the board modification plus shipped software. Both halves are settled. |
| **Experimental** | Reproduced once or twice. No burn-in, no second rig, or a known defect in the reporting. |
| **Attempted and failed** | A serious attempt exists; the result is negative, unstable, or unusable. |
| **No known lever** | The blocking mechanism is identified and nobody has produced a path past it. |
| **Not attempted** | Nobody in the record has tried it. |

## The board

| Capability | Status | How it is achieved | Read more |
|---|---|---|---|
| Compute unlock (SM throttle) | **Working (shipped)** | SEC2 Booter Load fired with an oversized signature buffer opens `FEAT_OVR_PLM 0x00823804`; the driver then writes `SS0 0x0082381c = 0x88888888` and `SS1 0x00823820 = 0x00000008` from the host | [Compute throttle](../unlock/compute-throttle.md) |
| Memory: 8 GB card to **64 GB** | **Working (shipped)** | `CFG1 0x009a0204 = 0x02779000`, `LMR 0x00100ce0 = 0x0000020B`, `targetFbBytes = 0x0000001000000000` | [Memory geometry](../unlock/memory-geometry.md) |
| Memory: 10 GB card to **40 GB** | **Working (shipped)** | `CFG1 = 0x02669000`, `LMR = 0x0000028A`, `targetFbBytes = 0x0000000A00000000` | [Memory geometry](../unlock/memory-geometry.md) |
| Memory: 10 GB card to **80 GB** | **Experimental** (the `80` branch itself: attempted and failed) | The `80` branch reports 81920 MiB then dies above ~40 GB. Separately, a driverless refire chain firing the coherent `CFG1 = 0x02779000` + `LMR = 0x0000028B` (decode `0x10000300`) verified real distinct memory with **no fold** to 77.5 GiB, and 72 GiB at stock boot timings. Limits: roughly one CUDA context per fire before Xid 154, ~79 % of peak bandwidth above the boundary. Not shipped, not installable | [The 80 GB problem](80gb.md) |
| PCIe **Gen2** (2.5 to 5 GT/s) | **Working (shipped)** | `0007-pcie-gen2.patch` (+ `0008` retrain): 25 Booter-routed register writes plus host BAR0 writes to the XVE / XP3G block. Merged to `master` 2026-07-29 in commit `2e0a2c02`; master's README now carries a `PCIe Gen 2 speeds | Working` row | [PCIe Gen2](../unlock/pcie-gen2.md) |
| PCIe **x16 width** | **Working (hardware)** | Solder 24 × 0402 220 nF X7R AC-coupling capacitors onto lanes 4-15 (designators C1100-C1350) | [Physical mods](../operations/physical-mods.md) |
| PCIe **Gen2 at x16 together** | **Working (hardware + shipped)** | Both of the above on one card; no extra step. Published captures: 6.63-6.67 GB/s on one card, 5.97 GB/s on each of four with zero AER errors | [Physical mods](../operations/physical-mods.md) |
| PCIe **Gen3 / Gen4** | **Attempted and failed** | No mechanism. `FUSE_PCIE_GEN3_DIS 0x00820580 = 0x1`; the supported-speeds vector clips at `0x00000006` | [Gen3 and Gen4](pcie-gen3-gen4.md) |
| **NVLink** | **No known lever** | Fuse-disabled (`0x00820684 = 0x7`). No FEAT_OVR entry exists for it, no code in any branch, no bridge ever seated | [NVLink](nvlink.md) |
| **ECC** | **No known lever** | Fused off (`OPT_ECC_EN 0x00820228 = 0x00000000`); `FBPA_ECC_CTRL` MASTER_EN is read-only | [ECC](ecc.md) |
| **P2P** (peer-to-peer DMA) | **Attempted, unresolved** | An out-of-tree P2P patch builds against a cmpunlocker tree; whether it does anything on a 170HX-only host is disputed | [P2P](p2p.md) |
| **Multi-card** | **Working (shipped, partially)** | The in-driver patch is per-GPU by construction; the multi-card *installer* is branch-only | [Multi-GPU](../procedures/multi-gpu.md) |
| **Driver backports** (595 / 590 / 580) | **Experimental** | `clanker/driver-port` branch, per-major-version patch directories. Source-verified, never boot-tested | [Driver versions](../procedures/driver-versions.md) |

## Row-by-row detail

### Compute unlock

Shipped, stable, and the only part of the unlock that survives a Function Level Reset (FLR).
`FEAT_OVR_PLM 0x00823804` sits on the always-on power island, so once opened it stays open;
`SS0` and `SS1` likewise persist. This asymmetry is the reason compute shipped before memory.

| Item | Value |
|---|---|
| SS0 `0x0082381c` | `0x88888888` (a locked card reads a per-die value, e.g. `0x53540175`) |
| SS1 `0x00823820` | `0x00000008` |
| `FEAT_OVR_PLM 0x00823804` | opened to `0xffffffff` (stock `0xffffff8f`) |
| Master kill fuse `0x008203f0` | `0x00000000`, unblown. This is why any of it works |
| Practical success test | `FEATURE_READOUT_1 0x00823818 == 0x00000000` |
| Survives FLR | Yes |

> [!WARNING]
> **Experimental**
>
> A MIG enable via bit 0 of `0x820840` was demonstrated with three corroborating `nvidia-smi`
> outputs and reported persistent, but it is not in the shipping tree and only the `1g.64gb`
> profile exists. INT8/IMMA throughput remains gated after the unlock for reasons nobody has
> explained.
>
> Second independent reproduction (610.43.03, 2026-09-13): enable, the sole `1g.64gb` instance,
> and reboot-survival confirmed. An A/B driver build shows `mig-unlock.patch` raises FP32 SGEMM
> under MIG ~5.5× (1.85 → 10.2 TFLOP/s) — FP32 throughput only; the INT8/IMMA gate is unaffected.
> Full report: [MIG](mig.md).

### Memory geometry

Geometry does **not** survive an FLR or a power cycle. The patched driver re-applies it on every
GSP boot, which is why the fix is a driver patch and not a one-shot tool.

| Quantity | 8 GB card (`10de:20c2`) | 10 GB card (`10de:2082`) |
|---|---|---|
| Stock capacity | 8192 MiB | 10240 MiB |
| Stock `CFG1 0x009a0204` | `0x02449000` | `0x02449000` |
| Stock `LMR 0x00100ce0` | `0x00000208` | `0x00000288` |
| Unlocked capacity | **65536 MiB** | **40960 MiB** |
| Unlocked CFG1 | `0x02779000` | `0x02669000` |
| Unlocked LMR | `0x0000020B` | `0x0000028A` |
| `targetFbBytes` | `0x0000001000000000` | `0x0000000A00000000` |
| Active FBPAs / bus width | 16 FBPAs (8 FBPs), 4096-bit | 20 FBPAs (10 FBPs), 5120-bit |

A third device ID, `10de:20b0`, is detected by `install.sh` but is **not** unlocked: the in-driver
gate `_kgspSec2PostblTimingEnabled()` accepts only `0x20C2` and `0x2082`.

> [!CAUTION]
> **80 GB on a 10 GB card is not a usable configuration**
>
> The `80` branch reports 81920 MiB (85,545,582,592 bytes) and a 77 GiB `cudaMalloc` succeeds,
> but at 80 GB, kernels touching more than roughly 40 GB cause fatal GPU loss, independent of
> power limit. Reported Xid codes include Xid 31 (described as harmless) and Xid 154 after CUDA
> memory tests; the dominant reported symptom is hangs. Xid 31 alone was suggested by a
> bystander and was not corroborated as *the* signature by the operator with the failing card.
> As actually built the branch programs CFG1 `0x02779000`, LMR `0x0000028A` and
> `fb_length 0x0000001400000000`, which is a three-way disagreement and is itself a candidate
> cause. The `0x0000028B` in that branch's `constants.yaml` is inert metadata: `build.sh` never
> reads the file. See [The 80 GB problem](80gb.md).

> [!WARNING]
> **Experimental: the coherent 80 GB set exists, but not as an install path**
>
> Separately from the branch, a clean-room refire script fired the *coherent* set
> (CFG1 `0x02779000` + LMR `0x0000028B`, L2 decode `0x10000300`) on 10 GB cards between
> 2026-07-23 and 2026-07-27, including at least one unmodded card. Dense tagged write/readback
> found **no fold** at 77.5 GiB, and 72 GiB passed at stock boot timings. The limits are real:
> roughly one CUDA context per fire before Xid 154, and about 79 % of peak bandwidth above the
> boundary. Two operators, no burn-in. Shipping master gives a 10 GB card **40 GB** and that
> remains the supported configuration.

### PCIe: speed and width are two different problems

> [!NOTE]
> **Do not conflate these**
>
> Gen1 to Gen2 is a **software** change to link speed. x4 to x16 is a **hardware** change to
> link width, caused by NVIDIA depopulating AC-coupling capacitors on 12 of the 16 lanes.
> Neither one affects the other. A capacitor mod alone gives Gen1 x16; the Gen2 patch alone
> gives Gen2 x4.

| Aspect | Stock, no unlock | With the unlocker | After capacitor mod |
|---|---|---|---|
| `LnkCap` | `0x00456101` | `0x00456102` | unchanged |
| `LnkCap2` | `0x00000002` | `0x00000006` | unchanged |
| `LnkSta` | `0x1041` (2.5 GT/s, x4) | `0x1042` (5 GT/s, x4) | 2.5 GT/s, **x16** |
| `nvidia-smi` cur/max/width | 1, 1, 4 | 2, 2, 4 | 1, 1, 16 |
| Measured bandwidth | ~0.85 GB/s (Gen1 x4) | ~1.71 GB/s (Gen2 x4) | 2.88 GB/s (Gen1 x16) |

Capacitor mod specification: **24 parts** (2 per differential pair × 12 depopulated lanes),
**0402, 220 nF (0.22 µF), X7R, ≥6.3 V**, designators in the **C1100-C1350** range. Confirmed part:
Taiyo Yuden `MAASJ105SB7224KFCA01`. The value comes from the NVIDIA A100 GA100-883 reference schematic
P1001-B02 page 3 ("IO: PCIe CONNECTOR"). Populating only 12 of the 24 yields x8, because PCIe
width negotiation falls back to the next legal width (16 to 8 to 4 to 1). An x8 result after a mod
means incomplete or bridged solder work, not a distinct hardware limit.

Gen2 is **absent from shipping master**: master carries patches `0001` through `0006` only, with
no `pcie:` block in `constants.yaml`. `0007-pcie-gen2.patch` exists on branches `debug-gen2`,
`Gen2`, `far` and `deced`; `0008-pcie-gen2-probe-retrain.patch` on `Gen2`, `far` and `deced`.

> [!WARNING]
> **Experimental**
>
> Gen2 does not train on every host. One Intel platform (ASUS W890 SAGE, Ubuntu 24.04) never
> reached Gen2 across two branches, an external fork, two slots, and both modded and unmodded
> cards, while an AMD HiveOS host reached it with no kernel command line changes at all. Gen2
> also does not train inside a VM under VFIO passthrough, and Thunderbolt 3 enclosures fail the
> *unlock itself*, not merely the link (`Booter Load 0x15 / 0xffff`,
> `RmInitAdapter failed! (0x62:0xffff:2119)`). Oculink works because it is essentially a riser.
>
> **This may already be fixed.** On 2026-07-27, the last Gen2 status change in the record, the
> maintainer published branch `deced` and stated the hardcoded `0a:00.0` PCI address was "the big
> bug that I think was causing all the issues", with VM passthrough named as the only known
> remaining case. No tester report came back before the corpus froze, and the wiki's own analysis
> holds that the file `deced` changed (`tools/retrain.sh`) is dead code on that lineage, which is
> an unresolved conflict. See [dead ends](../history/dead-ends.md).

> [!NOTE]
> **Gen2 at x16 needs no extra step**
>
> It is the capacitor mod plus the shipped Gen2 code, with nothing additional to do. First
> captured 2026-07-26: `PCIe GEN 2@16x`, `ocl_pcie_bw` 6.63-6.67 GB/s, nvtop TX 7.061 GiB/s.
> A second builder posted four cards across two board revisions with `lspci` captures and zero
> AER correctable or fatal errors over 90 minutes of continuous load. Long burn-in figures
> have not been published by anyone.

### Gen3 and Gen4

Attempted repeatedly and failed. Two fuses read `0x00000001` on both 170HX SKUs:
`FUSE_PCIE_GEN23_DIS 0x0082057c` and `FUSE_PCIE_GEN3_DIS 0x00820580`. The Gen2 patch attempts to
write `0x0082057c` to zero through the Booter and **the write fails**
(`status=0xffff rd=0x00000001`, followed by
`SEC2_DEBUG: PCIe xp3g booter FAILED to set OPT_GEN23`). Gen2 is reached instead by the `CYA_0` /
`LINK_CONFIG_0` / XP3G / `PRIV_MISC_1` overrides plus a root-port retrain. `0x00820580` has never
been written by anyone and no code path ever requests a target link speed above 2. Forcing the PHY
rate to a Gen3-capable `0x00340036` left the link at Gen1. Gen4 is additionally blocked on
equipment: the researcher who pursued it had no Gen4-capable host.

### NVLink

| Register | Value on 170HX | Meaning |
|---|---|---|
| `FUSE_NVLINK_DIS 0x00820684` | `0x00000007` | all three bits of the `[2:0]` disable field set |
| `STATUS_OPT_NVLINK 0x00820DB8` | `0x00000007` | read-only mirror agrees |
| `FUSE_NVLINK_DEFECTIVE 0x0082068C` | `0x00000000` | the silicon is intact; this is segmentation, not yield repair |
| `PTOP_SCAL_NUM_NVLINK 0x0002246C` | `0x0000000c` | the die carries the full 12-link GA100 complement |
| `CTRL_OPT_NVLINK 0x008209B8` | `0x00000000` | and `FUSE_EN_SW_OVERRIDE = 0x0`, so this path is inert |

There is no NVLink register anywhere in the `0x00823800`-`0x0082382C` feature-override block, so
the mechanism that unlocked compute does not apply. No branch contains NVLink code. Nobody in the
record has ever had a 170HX and an A100 NVLink bridge at the same time, so it is not even
established that the connectors align.

### ECC

Fused off with no telemetry. `OPT_ECC_EN 0x00820228` reads `0x00000000` on both 170HX units and
`0x00000001` on A100 SXM4 40G, A100 PCIe 40G, A100 PCIe 80G, A10, A5000, A6000 and the Drive A100.
`FBPA_ECC_CTRL 0x009a0470` reads `0x00000000` with `MASTER_EN` (bit 0) read-only, against
`0x00000041` on A100. The feature-override shadows exist and are populated
(`FEAT_OVR_ECC 0x0082380c = 0x00888888`, `_1 0x00823810 = 0x002aaaaa`, `_2 0x0082382c = 0x0000000a`)
but nothing has ever been written to them. `nvidia-smi -q` reports every ECC field as `N/A`.

The branch literally named `ecc` contains **no ECC code**: one commit, "Fixed dual geometry
support", and a standard 64/40 GB `constants.yaml`.

**Practical consequence:** an unlocked 170HX has no ECC counter, so silent corruption above the
real capacity ceiling never surfaces as an error statistic.

### P2P

`torch.cuda.can_device_access_peer` returns `False` for all 56 pairs on an 8-card host, ggml logs
show zero P2P activity, and a `MIG 1g.64gb` instance reports `P2P: No`. An out-of-tree P2P patch
was successfully built into a cmpunlocker tree, after which one tester reported it "doesn't seem
to take effect on the 170HX" and only helps when other GPU models share the machine. A second
report of "p2p + cmpunlock working" on the same day came from a rig that also contained two
RTX 3090s, which is exactly the mixed-model case, so the two reports may not conflict.
No `p2pBandwidthLatencyTest` matrix from a 170HX-only host exists.

### Multi-card

The in-driver patch reads `pGpu->idInfo.PCIDeviceID` on every GSP boot, so **a multi-card host is
unlocked correctly even by shipping master**, including a mixed 8 GB / 10 GB host. What master
lacks is installer support: its `install.sh` inspects only the first matching GPU
(`lspci ... | head -1`) and builds with a single profile.

The `multiple-cards` branch (tip `b1cb6d8`, committed 2026-07-19T05:41Z, which is 2026-07-18
local time in the author's `-07:00` zone) enumerates every
`10de:20b0|10de:20c2|10de:2082` device, builds per-BDF profile arrays, adds a third `mixed`
profile that sets `SKIP_GEOMETRY_REWRITE=1`, persists an inventory to
`/lib/modules/$(uname -r)/updates/cmpunlocker/gpu_inventory`, and adds a `verify.sh` that checks
each card by PCI bus ID. The same installer is folded into the `Gen2` branch. Neither has merged.

### Driver backports

| Tree | Versions | Boot-tested |
|---|---|---|
| Shipping `master` | `610.43.03` (default), `610.43.02` | Yes, both |
| `clanker/driver-port` | 12 in `driver/VERSION`, 5 in `constants.yaml` (`610.43.03`, `610.43.02`, `595.71.05`, `590.48.01`, `580.105.08`) | **No** |

Master hard-fails the build on any version outside its two-entry whitelist. The port branch adds
per-major-version patch directories: `580` (37,034 B), `590` (37,118 B), `595` (36,993 B) and
`610` (37,415 B, **byte-identical to master**). The branch's own `VERSION` and `constants.yaml`
disagree about which versions are even claimed, which is an acknowledged internal inconsistency.

> [!WARNING]
> **Experimental**
>
> No boot has been reported on any 595, 590 or 580 build. The patches apply cleanly and the
> sizes are plausible; that is the entire basis. One tester per branch reporting
> `dmesg | grep SEC2_DEBUG` and the `POST-BooterLoad verify` line would settle it.

## Secondary capabilities

| Capability | Status | Notes |
|---|---|---|
| MIG | **Experimental** | Enable via bit 0 of `0x820840` demonstrated and reported persistent; only the `1g.64gb` profile exists, `-cgi 9,3g.20gb -C` returns `Invalid Argument`. The device-info table enumerates only GR0; an inferred syspipe mask reads `0xfe` and ignores host writes ([MIG](mig.md)). Not upstreamed. |
| Resizable BAR | **Not attempted** | The card advertises a Physical Resizable BAR capability, reportedly limited to 64 MiB. Master deliberately clamps the BAR0/PRAMIN window to the 8 GiB stock offset for both device IDs. |
| SR-IOV | **Not attempted** | No SR-IOV extended capability appears in the archived `lspci -vvv` capture, which is suggestive but was captured for other purposes. |
| NVENC | **No known lever** | Probably a silicon absence: GA100 generally lacks NVENC hardware. NVDEC is present. |
| Windows | **Not attempted** | The unlock is a patch to the Linux open kernel modules and has no Windows analogue. A driverless Python compute-only attempt is the only credible cheap experiment. |
| VM passthrough | **Working (with caveats)** | Compute and memory unlock work under Proxmox, but the guest must use **SeaBIOS, not UEFI/OVMF**; UEFI produces `RmInitAdapter` failures that look like the exploit simply not working. Gen2 does not train in a guest. |
| Thunderbolt 3 eGPU | **Attempted and failed** | Booter Load fails outright. Use bare metal or Oculink. |

## Where the frontier actually is

Ranked by how close each is to falling, the live problems are: **Gen2 merged to master**
(blocked on cleanup, not on knowledge), **the coherent 80 GB set carried by a driver build rather
than a fire script**, and with it the Xid 154 one-context-per-fire limit that is now the real
80 GB blocker rather than the fold,
**Gen3 via `0x00820580 = 0`** (never attempted, but the prior is low: the neighbouring
`0x0082057c` write through the same `xp3g` table is observed to fail on silicon),
**P2P measured on a 170HX-only pair**, and **ECC** (blocking mechanism identified, no lever found).
NVLink is the highest-value unknown with nothing tractable on the table.

Every one of these, with what has been tried and what evidence would settle it, is in the
[open-problem register](open-questions.md).
