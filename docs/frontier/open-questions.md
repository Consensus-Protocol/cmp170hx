# The open-problem register

**What this page covers.** Every unsolved problem on the CMP 170HX, aggregated from all 24
adjudicated domain documents, deduplicated, and ranked by tractability rather than by importance.
For each item: what is wanted, what has been tried, why it failed, what evidence would settle it,
and the most promising next step.

The single most useful thing to know is that **a large fraction of the open list is cheap**. Six of
the items in Tier 0 need no hardware at all, only a header file, a hash, or a `--dry-run`. Another
dozen in Tier 1 need one card, one reboot, and a posted log. The genuinely hard problems (NVLink,
ECC, Gen3) are hard for identified reasons, and those reasons are stated below rather than
hand-waved. 80 GB used to sit in that group and no longer does: the fold is explained and gone, and
what is left is narrower (see item 1.1).

Items that closed during the archived period are kept in place, marked closed, with the result
recorded, rather than deleted. Nothing below is still open unless it says so.

For current capability states rather than open problems, see the
[status board](status-board.md).

## How items are ranked

| Tier | Cost to close |
|---|---|
| **Tier 0** | No hardware. A file, a header, a hash, an offline dry-run, or a code change. |
| **Tier 1** | One card, one boot or one command. A log posted afterwards settles it. |
| **Tier 2** | One card plus a rebuild, a controlled A/B, or an instrumented run. |
| **Tier 3** | New capability: equipment, hardware nobody has, or a technique nobody has built. |
| **Tier 4** | Measurement disputes. Not unknown so much as never measured under one methodology. |
| **Unanswerable** | Nothing in the record or the code can settle it. |

---

## Tier 0: settleable without touching a card

### 0.1 The LMR magnitude field width

**Wanted:** the width of `LOWER_MAG` in `NV_PFB_PRI_MMU_LOCAL_MEMORY_RANGE` on GA100: 6 bits at
`[9:4]` or 7 bits at `[10:4]`.
**Tried:** arithmetic only. Under a 6-bit field the rule `size_MiB = MAG[9:4] << SCALE[3:0]` is
exact for all five values in real use (`0x208`, `0x288`, `0x20B`, `0x28A`, `0x28B`), and `0x50A`
decodes to 16384 MiB, not 81920 MiB. Under a 7-bit field `0x50A` would be `80 << 10` = 81920 MiB.
**Why it is open:** nobody has read `dev_fb.h`.
**Evidence that settles it:** the header definition. This is a lookup, not an experiment.
**Why it matters:** it is the last thing standing between the `0x28B`-versus-`0x50A` dispute and a
clean answer, and that dispute feeds directly into the 80 GB retry (item 1.1).

### 0.2 Do the seven unverified point releases in the port branch apply?

**Wanted:** either fuzz-match confirmation or removal from the whitelist for the seven driver
versions listed in `driver/VERSION` on the `clanker/driver-port` branch that no `constants.yaml`
entry backs.
**Tried:** nothing. The risk was found by code reading.
**Next step:** download each tarball and run `patch -p1 --dry-run` against the matching major
version patch directory. Purely offline and mechanical.

### 0.3 Is `booter_load` really unchanged across driver 535 to 610?

**Wanted:** confirmation of a claim that is used operationally ("extract once"), with 580-named
binaries fed to a 610-era chain.
**Tried:** asserted twice, never shown. No hashes across versions appear anywhere.
**Evidence that settles it:** SHA-256 of the decompressed `IMAGE_PROD` entry from
`g_bindata_kgspGetBinArchiveBooterLoadUcode_GA100.c` in 535.x, 580.x and 610.x.
Related and equally cheap: hash `IMAGE_DBG` against `IMAGE_PROD` in the same bindata archive,
which settles whether a debug-derived ROP chain is guaranteed to work on production silicon.

### 0.4 Fix the `0008` retrain success predicate

**Wanted:** a driver log that tells the truth about Gen2. `LnkSta 0x1042` is a genuine Gen2 link,
but `0008` includes `PCI_EXP_LNKSTA_DLLLA` (bit 13) in its success test and that bit can never be
set, because the GPU's `LnkCap` bit 20 (DLL Link Active Reporting Capable) is 0.
**Consequence:** the patch prints failure on a working link, which has already misled one
downstream analysis into concluding `0008` "runs too late".
**Next step:** drop the DLLLA term, or make it conditional on `PCI_EXP_LNKCAP_DLLLARC`, or read
`LnkSta` from the upstream bridge instead of the endpoint. One-line change in
`0008-pcie-gen2-probe-retrain.patch`.

### 0.5 Tooling reporting gaps

Three self-contained additions that need no hardware to write:

- Add the five PLM addresses the shipping unlock actually manipulates to `probe.sh`:
  FBPA PLM `0x009a0148`, WPR PLM `0x001fa7c4`, WPR_CFG PLM `0x001fa7cc`, WPR2 lo/hi
  `0x001fa824` / `0x001fa828`, and MMU LMR `0x00100ce0`. The existing 120-register catalog predates
  the unlock and covers none of them.
- Add a PCIe check to `verify.sh`. It does not check link speed on any branch, **including the
  Gen2 lineage**, so the branch whose README claims Gen2 "Working" ships a verifier that cannot
  see Gen2.
- Pin or checksum the `open-gpu-kernel-modules` tarball. `build.sh` fetches it with
  `curl -L --fail` and caches it with no verification anywhere in the tree.

### 0.6 Merge blockers for Gen2 and multi-card

**Wanted:** Gen2 and multi-card support in the default install.
**Blockers visible in the tree:** `0007` is a large debug-instrumented hunk logging at
`LEVEL_ERROR` throughout; `tools/retrain.sh` is dead code on the current Gen2 lineage (`Gen2`,
`far` and `deced` all `rm -f` it at install time, and only the earlier `debug-gen2` ever installed
it; its hard-coded BDF was fixed only on `deced`); `constants.yaml` omits five registers the
mechanism depends on.
**Next step:** the `multiple-cards` installer changes (`b1cb6d8`) are self-contained and could land
alone, independent of the Gen2 PCIe patches. Master's patch `0001` already bakes in both
geometries, so the `mixed` profile works without any driver change.

---

## Tier 1: one card, one boot

### 1.1 What still limits 80 GB now that the fold is gone

**Largely answered on the driverless path; the in-driver build has still not been rebuilt with it.**
**What was open:** whether the `80` branch's fold at exactly 40 GiB came from the incoherent LMR or
from the LTC decode. The branch as compiled programs CFG1 `0x02779000` (tier `0x77`, 4096 MiB per
FBPA, 81920 MiB across 20 FBPAs) against LMR `0x0000028A` (40960 MiB) and `fb_length 0x1400000000`
(80 GiB). Three layers, three different answers, and the fold boundary matched the LMR exactly.

**Answered on the driverless path, 2026-07-25.** A fire that combined LMR `0x028b` with
CFG1 `0x02779000` and the L2 decode at `0x10000300` came up with **79.3 GiB total, 79.0 GiB free**
and a **77 GiB** buffer that did *not* fold: the offset sweep read real bandwidth past the old
40 GiB collision point, and the run was summarised in-channel as "80GB seems to be verified in
tests". A 72 GiB no-fold run followed on 2026-07-27 at stock boot timings. So the coherent-LMR
hypothesis is supported and the LTC decode is not an absolute wall.

**What is still open** is narrower than this item's original framing:

- Bandwidth above 40 GiB sits at roughly 79% of peak rather than the ~98% seen at low offsets.
- Only about one CUDA context survives per fire before Xid 154.
- Nobody has reproduced any of it from a compiled driver. Setting
  `lmrValue = 0x0000028BU` in `driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` **and**
  `LMR="0x0000028B"` in `driver/build.sh` (not just `constants.yaml`, which the build never reads)
  remains uncompiled and unbooted.

See [the 80 GB problem](80gb.md).

### 1.2 `RMPcieLinkSpeed=0x1` or `0x2`?

**Wanted:** the correct modprobe line for Gen2.
**State:** both spellings ship. `debug-gen2/install.sh:191` and `Gen2/install.sh:280` write
`NVreg_RegistryDwords="RmForceEnableGen2=1;RMPcieLinkSpeed=0x1"`; `far/install.sh:280` and
`deced/install.sh:280` write `0x2`, introduced by commit `8854d3e` "Remove clamp link to Gen1".
Both readings are internally coherent depending on whether the key means "clamp to gen N" or
"enable up to gen N", and the branch whose README claims Gen2 works ships `0x1`.
**Next step:** boot the same card and kernel three times (no key, `0x1`, `0x2`) and post `LnkSta`
each time. Cheap and decisive.

### 1.3 Do the 595 / 590 / 580 driver ports boot?

**Tried:** the branch was announced 2026-07-21 with an explicit call for testers; nothing came back.
**Why it stalled:** it needs an owner willing to downgrade a working card.
**Evidence that settles it:** one tester per branch reporting `dmesg | grep SEC2_DEBUG` and the
`POST-BooterLoad verify` line. That single line answers it.

### 1.4 Is `610.43.02` or `610.43.03` more reliable?

Asked directly in-channel and never answered. Successful unlocks exist on both; `610.43.03` is
merely first in `driver/VERSION`.
**Next step:** collect the `driver_version` metadata file plus the SEC2 PLM-open success rate from
the existing installed base and compare.

### 1.5 Is MIG usable on an unlocked card?

The MIG-relevant registers are populated and readable (`FBHUB_MEM_PART_BOT 0x00100b88`,
`MID 0x00100b8c`, `BOUNDARY_CFG0 0x00100b90`, `SYSMEM_HSHUB_CONNECTION_CFG 0x00100b98`). These FBHUB
memory-partition registers describe the memory side and do not settle the compute-side profile
limit. New evidence on 610.57.04 places that limit at device enumeration: the hardware device-info
table lists only GR0, and an inferred syspipe-disable mask (`0x008203a4`) reads `0xfe` and does not
take host writes. The register-to-engine mapping is not yet validated against an A100. See
[MIG: root cause](mig.md#root-cause-evidence-for-a-hardware-enumeration-limit-syspipe-fuse-attribution-provisional).
Separately, a MIG enable via bit 0 of `0x820840` was demonstrated and reported persistent.
**Already reported once:** on an unlocked card `nvidia-smi mig -lgip` lists exactly one profile,
`MIG 1g.64gb` (63.00 GiB, 70 SMs), and `nvidia-smi mig -cgi 0` creates it, while a standard A100
profile (`-cgi 9,3g.20gb -C`) returns `Invalid Argument`. So MIG turns on but cannot partition the
card as shipped.

**Reproduced independently (2026-09-13, driver 610.43.03, one CMP 170HX at 64 GB):** `-mig 1`
enables and survives a reboot; `nvidia-smi mig -lgip` lists exactly `MIG 1g.64gb`; `-cgi 1g.64gb -C`
creates the GPU and compute instance; every standard A100 profile (by name and by ID) still returns
`Invalid Argument`. An A/B driver build isolates the patch's effect: on the base unlock an FP32
cuBLAS SGEMM (8192³) on the instance runs at 1.85 TFLOP/s, and with `mig-unlock.patch` at
10.2 TFLOP/s — a ~5.5× throughput gain consistent with widened compute-instance capacity. This
measures FP32 throughput only: it does **not** establish an active-SM count (`-lgip` reports 70 SM
in both builds) or address the separate INT8/IMMA gate. Full commands, outputs and A/B method:
[MIG](mig.md).
**Next step:** the second-card reproduction is now done (above); what remains is to find where RM
builds the GA100 GPU-instance profile list so more profiles can be added. The MIG enable holds
across reboots on 610.43.03, and the maintainer offered to merge that patch.

### 1.6 Does P2P do anything on a 170HX-only host?

**Tried:** the out-of-tree P2P patch builds into a cmpunlocker tree. One tester reported no effect
on 170HX-only. A same-day positive report came from a rig that also held two RTX 3090s, which is
the mixed-model case the negative report says is the only one that works.
**Evidence that settles it:** a `p2pBandwidthLatencyTest` connectivity matrix from a 170HX-only
multi-card host, with and without the layered patch. The test is cheap and the result is
unambiguous.
**Caveat:** everyone involved agrees P2P is bandwidth-bound and buys little at Gen1 x4.

### 1.7 Long-term stability at Gen2 x16

Reproduction is settled: two rigs, five cards, two board revisions, `lspci` captures and zero AER
errors over 90 minutes of continuous four-card load. See the
[field report](../operations/physical-mods.md#field-report-four-cards-at-gen2-x16).

What is still missing is duration. Nobody has published error counters after days of sustained
load, which is the only remaining way a marginal AC-coupling network would show itself.
**Next step:** anyone running a capacitor-modded card at Gen2 x16 posts
`cat /sys/bus/pci/devices/<bdf>/aer_dev_correctable` after a week of uptime, alongside
`lspci -d 10de:20c2 -vv | grep -E 'LnkCap:|LnkSta:'`.

### 1.8 What does `0x008200FC` read, and is it writable?

Three incompatible observations: `0x000003FF` with the write failing and direct host `writel` also
capped at `0x3FF` (2026-07-09); `0xffffffff` across the whole Ampere lineup (2026-07-16); and
`PLM[8] OPT_PLM(0x8200fc) attempt=0 status=0xffff reg=0xffffffff`, reported as success, from the
nine-PLM branch tool (2026-07-23/24). The same register carries two names: `OPT_PLM` in branch
code, `FUSE_SS_PLM` in the clean-room tooling.
**Next step:** read `0x008200FC` on a cold-booted card before any unlock, then after each of the
nine PLM opens, on the same unit.

### 1.9 The RAMCFG re-strap result (closed)

> [!TIP]
> **Closed 2026-07-25: confirmed negative, as predicted**
>
> The RAMCFG resistors on a 10 GB card were moved to `HHLLH`, the 8 GB stock strap pattern. The
> card booted, and `probe.sh` read `FBPA_CFG1_BROADCAST` at `0x009a0204` still equal to
> **`0x02449000`**, the 10 GB value. Summarised in-channel as "the resistor change had no effect".
> This is exactly the **predicted outcome from code**: the driver keys geometry off the PCI device
> ID, not off the strap. Worth noting the card booted with the L2 decode already at `0x10000300`.
> Physical RAMCFG re-strapping is a dead end for capacity.

### 1.10 Does the eviction-test fold reproduce on the shipping geometries?

The offset sweep showing a uniform 79%-of-peak plateau above the 8 GiB boundary was only ever run
on the abandoned 80 GB configuration. Shipping master clamps BAR0/PRAMIN to exactly 8 GiB for both
device IDs, which is suggestive, but PRAMIN is a CPU aperture and the sweep was a device-side
memset.
**Next step:** repeat the identical offset and chunk sweeps on a shipping 8 GB to 64 GB card. If
the step reappears at 8 GiB it is a geometry effect; if not, it was specific to the over-fire.

### 1.11 The uBatch cliff

A CMP 100-210 with 68 of 84 SMs showed a **3.04x prompt-processing gain** from one `n_ubatch` flag
tuned just below the SM count. Nothing equivalent has been run on a 70-SM 170HX.
**Next step:** `llama-bench` with `n_ubatch` swept 48 to 80 on a compute-unlocked card. Cheapest
untested performance lead in the corpus.

### 1.12 Log the throttle onset

Four temperature figures coexist: throttling observed at 80 C, community reports of 85 C, a driver
report of `GPU Max Operating Temp 85 C` / `GPU Slowdown Temp 95 C`, and an account of throttling
above 100 C when cooling fails. These may all be true at different layers, but nobody captured a
throttle log.
**Next step:** log
`nvidia-smi --query-gpu=temperature.gpu,clocks.sm,clocks_throttle_reasons.active` through 75-95 C.

---

## Tier 2: one card plus a rebuild or a controlled A/B

### 2.1 An intermediate capacity on the 10 GB card

**Wanted:** something between the stable 40 GB and the broken 80 GB. Asked directly in-channel and
answered "I don't believe its been tried."
**Why the obvious route does not work:** the per-channel tier is coarse (`0x44`/`0x66`/`0x77` =
512 / 2048 / 4096 MiB), so intermediate sizes are unreachable by tier alone.
**Next step:** fix CFG1 at tier `0x77` and clamp the driver-visible size through `targetFbBytes` to
48, 56 or 64 GB, then run the write-all-then-read-all fold test at each rung. One constant change
and one reboot per rung. It is also the decisive falsifier for the bad-stack binning theory.

### 2.2 Broadcast versus per-FBPA CFG1 writes

The shipping driver writes CFG1 only to the broadcast alias `0x009a0204`. The one reported
phantom-free readback came from writing all FBPAs individually. Nobody has instrumented whether the
FBPA broadcast is a genuine PRI priv-ring hardware mechanism.
**Next step:** with the FB-geometry PLMs open and no devinit, write only the broadcast and read all
24 per-FBPA CFG1 mirrors at `0x00900204 + n*0x4000` plus `CSTATUS_RAMAMOUNT` at
`0x0090020C + n*0x4000`, then repeat with per-FBPA writes and compare.

### 2.3 Which PLM actually gates the LMR?

**Wanted:** the privilege mask that guards `0x00100CE0`, so LMR could be written from the host
without an HS fire. A candidate FBHUB table was posted and then disclaimed by its own author.
**The shortcut nobody took:** the shipping driver *does* write LMR successfully from the host after
opening exactly four PLMs, so **one of those four already gates it**. A four-way ablation of the
shipping `plmTable[]` identifies which.

### 2.4 What the leading digit of CFG1 means (closed)

> [!TIP]
> **Closed 2026-07-27: bit 29 halves the tier**
>
> The experiment was run. Writing CFG1 `0x22779000` in place of `0x02669000` on a card with 20
> live FBPAs **halved the `0x77` tier** to 2048 MiB per FBPA, giving 20 x 2048 = 40960 MiB (40 GB).
> Per-FBPA `CSTATUS` stayed at `0x0800` and LMR stayed at `0x0000028a` across the change, so the
> capacity move came from CFG1 alone.
>
> A second effect showed up in the timing block: `HBM_CFG0` at `0x9a038c` went `0xa7` to `0xa6`,
> the dual-rank bit, and 6 of 7 timing registers moved to their strap-5 values. So the leading
> digit is not a pure capacity divider; it re-selects a rank configuration and drags the timing
> strap with it. The candidate VBIOS strap entry `00 90 77 22` carrying the same odd `2` is
> consistent with that reading.
>
> The ROP path may be required for the write; that part was reported as not entirely certain.

### 2.5 Gen3 through the same table that attempts the Gen2 fuse

**Worth one boot, but the prior is low.** The 23-entry `xp3gTable` does contain
`FUSE_PCIE_GEN23_DIS 0x0082057c`, but that write is observed to *fail* on silicon
(`status=0xffff rd=0x00000001`, then `SEC2_DEBUG: PCIe xp3g booter FAILED to set OPT_GEN23`), so
there is no proven write path on this fuse bank to extend to `0x00820580`. Gen2 is reached by the
`CYA_0` / `LINK_CONFIG_0` / XP3G / `PRIV_MISC_1` overrides plus a root-port retrain, not by that
fuse write. `FUSE_PCIE_GEN3_DIS 0x00820580` reads `0x00000001` and **has never been written by
anyone**, and no code path ever requests a target link speed above 2.
**Next step:** add `0x00820580 = 0` to the same table, then request TLS = 3 and read `LnkCap2`. If
it reaches `0x0000000E` the contiguity argument holds; if it stays at `0x00000006` the Gen3 fuse is
enforced independently downstream, which is currently the better-supported reading (forcing the PHY
rate to a Gen3-capable `0x00340036` left the link at Gen1).

### 2.6 The second feature-override group

`0x823830` through `0x82383C` read `0xbadf5040` from PL0 and return real values on an HS read. No
manual PLM covers the group and an HS write-then-readback has never been performed. Related and
also untested: `0x823b00` as the candidate PLM for the row remapper at `0x823824`.

### 2.7 Isolate SS1

The claim that "SS1 nerfs 64-bit compute" is an untested belief that happens to sit next to a
correct FP64 measurement. SS1 has been in the recipe since 2026-07-14 and always shipped alongside
SS0.
**Next step:** a one-line build with the `0x00823820` write removed, then re-run the OpenCL FP64
test. Expect 6.421 TFLOP/s if the belief is wrong, something near 0.19 if it is right.

### 2.8 Why INT8/IMMA is still gated

An unlocked card does INT8 at 44.1 TOPS, roughly 3.7x *slower* than its FP16, where an A100 does
INT8 at about 2x FP16. The IMLA fuses read the same `0x5` as the FMA ones and SS0 sets their
override nibbles identically, yet the measured rate does not follow. Separately, the raw MMA path
measures 335 TOPS while the library path (torch, cuBLAS, OpenCL) sits at 43-48.
**Next step:** an explicit `CUBLAS_COMPUTE_32I` GEMM with INT8 inputs against the raw MMA figure,
plus a dump of `0x00823818` alongside a per-datatype microbenchmark to check whether the
*effective* IMLA fields really are zero.

### 2.9 Residual compute gates

- `SM_ISSUE_RATE_MODIFIER 0x00504204` reads `0x00000005` on a fully unlocked card and is not touched
  by the unlock. Writing it to 0 on an unlocked card and re-running the benchmark suite has never
  been done, though the register is host-writable at PL3 and the write primitive exists.
- `SKED_UNK54 0x00407054` is the most-referenced undocumented SKED register in the GSP firmware
  (13 references) and the only register in the 13-card cohort that is non-zero on the 170HX and zero
  on both A100 and RTX 3090. Nobody has write-tested it. The driver clears it during GR init, so it
  is only observable pre-driver.
- `FEATURE_READOUT_1 0x00823818` has no known field layout. A naive nine-by-three-bit unpack of the
  stock `0x016DB6ED` does not match the fuses. Sweeping single nibbles of SS0 on an unlocked card
  and watching which bits move is a direct bit-mapping experiment.

### 2.10 Training status versus the "untrained memory" theory

`FBPA_TRAINING_STATUS 0x009a0974` reads `0x00000000` on every card, **including an unlocked card
already carrying the 64 GB CFG1 value**, yet the leading explanation for 80 GB instability is that
the upper region is untrained and without timings. Nobody has correlated `TRAINING_STATUS` with an
actual crash trace.
**Next step:** read it immediately before and after a crash at the over-provisioned tier, and diff
the `TIMING*_GEN` shadows between a stock boot and an unlocked boot.

### 2.11 Retention versus address decode at 80 GB

Two mechanisms are proposed for the >40 GB failure and they predict different failure geometry. A
retention failure scatters errors by time and address across the whole upper region; a decode fold
produces exact aliasing at a power-of-two boundary. The observed fold sits at **exactly 40 GiB**,
which is suspiciously exact for a retention problem, and doubling refresh landed successfully on all
20 live FBPAs without fixing the instability while costing roughly 40% of bandwidth.
**Next step:** run the write-all-then-read-all fold test at 80 GB geometry with stock timings,
single context, on a freshly cold-booted card, and report both pass/fail and the address
distribution of failures.
**Related and unexplained:** `cuda_memtest` passes over all 80 GB once immediately after a reboot
and fails on every retry. That reboot dependence is the most concrete remaining lead.

### 2.12 Recoverable FBPs

Some cards show `FBP_DISABLE 0x852` against `FBP_DEFECTIVE 0x840`, two extra disabled-only bits. The
one 10 GB card actually dumped in-channel had `DISABLE == DEFECTIVE == 0x840`, and the
software-override experiment showed the effective mask does not move even from HS.
**Next step:** run the eight-register read on a card whose masks differ (an 8 GB-class card), attempt
re-enable under CPU-RM with GSP disabled, and check whether writes to `0x00820364` survive an FLR.
**Caveat:** whether "disabled but not defective" silicon is recoverable at all is itself unproven.

### 2.13 IOMMU and the Gen2 reliability story

Directly contradictory reports exist. Multiple testers went from "still PCIe 1" to success after one
grub change and the branch installer now sets IOMMU automatically; an independent setup script lists
`iommu=pt` / VT-d among things "tested and confirmed unnecessary", and its confirmed hosts made no
grub changes at all.
**Evidence that settles it:** a matrix of {IOMMU off, on, pt} × {Intel, AMD} × {userspace hammer,
in-driver `0008`} on identical software. Plausible but undemonstrated reconciliation:
`iomem=relaxed` matters only for the userspace `mmap`-based retrain and IOMMU mode only on some
chipsets.

### 2.14 Reboot persistence of Gen2

`nvidia-smi` reports `2, 2` immediately after `install.sh` and `1, 1` after reboot on some systems,
reproducibly, with re-running `install.sh` restoring it each time.
**Working hypothesis:** persistence depends on whether the *patched* module is actually loaded at
boot rather than only live-patched by the installer.
**Next step:** on a failing system, check `modinfo` and `dmesg` after reboot for the patched module
and for `SEC2_DEBUG: PCIe` lines before blaming the retrain.

### 2.15 The FLR survival map has one unwithdrawn dissent

Geometry reverting on FLR was reported as measured "every time" (2026-07-12), then reversed by the
same researcher the next day, then re-established by an FLR readback table and a survival map. The
adjudication leans strongly to "FLR reverts", but the reversal was never formally withdrawn.
**Next step:** one clean FLR readback of CFG1, LMR and CSTATUS with no driver loaded, published with
the card ID.

---

## Tier 3: needs new capability, equipment, or hardware nobody has

### 3.1 ECC

**Wanted:** error correction, which would materially change the risk picture and might make
marginal geometries usable.
**Tried:** nothing on hardware. Asked directly on 2026-07-20, answered "Not yet."
**Blocking mechanism:** `OPT_ECC_EN 0x00820228 = 0` and `FBPA_ECC_CTRL 0x009a0470` with `MASTER_EN`
read-only. It is not known whether `MASTER_EN` is fuse-gated or PLM-gated, and those imply very
different ceilings.
**Cheapest first experiment:** the `FEAT_OVR_ECC` family (`0x0082380c`, `0x00823810`, `0x0082382c`)
is writable in principle once `0x00823804` is open, which the shipping patch already does. Test
whether writing it has any effect at all post-DEVINIT.
**Partly resolved prerequisite:** whether the HBM stacks carry ECC provisioning at all. One reading
of the fuse table, never checked against a datasheet, says ECC on GA100 is a feature of the HBM2
stack taken as in-band capacity: A100 per-FBPA `CSTATUS_RAMAMOUNT` reads `0x07ff` / `0x0fff` where
consumer parts read `0x0800` / `0x1000`. If instead it is a vendor QA binning property, the
question is not meaningful. See [ECC](ecc.md).

### 3.2 NVLink

**Wanted:** the actual unlock. **Summary as of 2026-07-20: "Still unsolved, a bit harder as there's
no fuse mask."**
**Both override paths are closed:** `CTRL_OPT_NVLINK 0x008209B8` is inert because
`FUSE_EN_SW_OVERRIDE = 0x0` and `FUSE_DIS_SW_OVR = 0x1`; the FEAT_OVR route is closed because the
`0x00823800`-`0x0082382C` block contains no NVLink register at all.
**Nothing tractable is on the table.** The three cheapest things that would at least produce data:

1. A read-write-read probe of `0x008209B8` and `0x00820820` on an expendable card, then re-read
   `STATUS_OPT_NVLINK 0x00820DB8`. The expected result is that the write is dropped and status
   stays `0x00000007`. **That negative is worth recording**, because the corpus currently cannot
   even say it was tried.
2. High-resolution photographs of a de-shrouded 170HX at designators `R234`, `R236`, `R237`, `R238`,
   `R976`, `R1024`, `R1029`, `R1030`, plus continuity from the NVLink edge fingers to BGA balls
   `F1` and `G1`. This settles the standing dispute over whether the PCB area is populated at all,
   which decides whether a fuse bypass would even be useful. Note `R237` is marked **NP** in the
   A100 schematic itself, so at least one is expected absent on a genuine A100 too.
3. Seat an actual A100 bridge. Nobody in the record has ever had a 170HX and a bridge at the same
   time, so it is not even established that the connectors align.

Interposer fabrication was discussed (connector sourcing, Megtron laminate, non-standard card
orientation) and is pointless before item 3 returns a positive.

### 3.3 Gen4, and the DevInit five-byte edit

The PCIe Gen1 lock is 5 bytes across 3 devinit sites **inside** the VBIOS MAC range, so the ROM
route is closed without forging the MAC. Whether that edit alone would restore Gen4 given a flash
that could be re-signed has never been tested; the "firmware-only patch insufficient" conclusion is
an inference from fuse values, not the outcome of an attempted patch. Additionally, the researcher
who pursued Gen4 had **no Gen4-capable host**, so it is blocked on hardware access before it is
blocked on technique.
Named but untried alternatives: a wire-level retimer interposer (Astera Aries, TI
DS160PR810-class) forging TS1/TS2 Rate-ID, and a GSP patch diverting where GSP-RM consumes the
`0x57c` / `0x580` fuse reads (jump table at `0x5D55834` in 470.42.01 `gsp.bin`, `0x4DD9B00` in
580.105.08 `gsp_tu10x.bin`).

### 3.4 HBM timings and MRS replay

**Wanted:** correct timings for the over-provisioned region, and the ability to change the memory
strap without physically removing the cooler for a reflash.
**Why writing `TIMING*` registers does nothing:** `CONFIG0.USE_TIMING_REGS` (bit 31 of `0x9A0290`)
is **0**, so the controller ignores the raw values and uses internally generated ones.
**Why MRS replay fails:** the published paper reports that *every agent it could drive* (host, SEC2
in non-secure mode, SEC2 in HS, PMU, GSP in non-secure mode) failed to issue a mode-register-set
command the die consumed. Writes appear to be dropped on a source-identity basis. The only agent
that visibly drives the die is the FB Falcon running its own code at boot.
**Mechanically working, then blocked:** baseline MR1 `0x00100093`, MR2 `0x002000cf`,
MR3 `0x003000ea`; derived strap-7 MR1 `0x0010009b`, MR2 `0x00200029`, MR3 `0x003000ef`; replay back
to strap 4 reproduced the boot values exactly. The driver then refuses with
`RmInitAdapter failed! (0x62:0x40:2674)`.
**Next step named in-channel and never attempted:** patch the driver to tolerate the changed state
rather than trying to make the state look untouched.
**Prerequisite for everything here:** dump the HBM self-report (IEEE 1500 block,
`0x009a3cb4`..`0x009a3cc8`), the FBPA training-result registers and the MR registers from a
known-good 64 GB card and from a 10 GB card.

### 3.5 The L2 / LTC decode wall

A managed-memory atomic kernel paging past 40 GB flips the L2 decode at `0x17E2A0` from
`0x70000300` (2 GB per channel) to `0x10000300` (4 GB per channel) through the UVM atomic-fault
path. The workload was 2 MiB-stride atomics across 46 GiB, roughly 23,552 atomics in 0.3 s. The
decode holds while the workload is live and reverts the moment the context tears down, and the
triggering fault poisons the context (CUDA error 719, then Xid 45, then Xid 154). A one-byte driver
mod can hold it there without the workload.
**Why this is not the answer:** flipping it does not fix the >40 GB problem, which is
"downstream, unfortunately". What "downstream" means was never pinned down, and **that is the gap**.
**Next step:** find a non-faulting trigger for the same lazy re-enable, and characterise what
downstream actually blocks.

### 3.6 Recovering a csecret

Three indices map to three capabilities: **secret(6)** decrypts the ECB firmware blobs (would yield
121.7 KB of plaintext firmware plus Booter code); **secret(2)** forges the content MAC, which would
open the 2 CFG1 bytes and the 5 PCIe devinit bytes at once, i.e. the entire 7-byte VBIOS
restriction; **secret(0)** is the debug bypass enabling a HULK cert with `SKIP_VBIOS_SIG`.
**No csecret has been recovered.** All three are differential-fault-analysis targets requiring
voltage-glitch hardware. An instant offline key verifier exists from the all-`0xFF` known-plaintext
oracle, so a DFA campaign needs no flash cycles for validation. Very high effort.

### 3.7 VBIOS edits that need a hardware programmer

**Updated 2026-07-25: `nvflash` is not blocked outright.** A *patched* `nvflash` (omgvflash-style)
wrote a full stock 8 GB image onto a 10 GB card and then flashed it back, with no leak and no
programmer. Two things still fail: a **strap-edited** image trips a host-side image check during
flashing, and the foreign image does not boot. There was no reboot between the two flashes, so
treat the write itself as suggestive rather than definitive; see [vbios.md](../hardware/vbios.md).

What this does *not* establish is whether an **unsigned-tail** edit survives that image check.
Nobody has tried one. So a CH341A with a SOIC clip (and a 1.8 V adapter), cooler removal and a
repaste remains the assumed route for the three edits below, but the cheaper experiment now is to
attempt the tail edit with the patched `nvflash` first:

- **250 W to 300 W power limit at `0x45E45`.** The 3-byte field reads `90 D0 03` and should become
  `E0 93 04`. It sits inside the 15,616-byte unsigned FwSec tail at `0x43A00`-`0x47700`, so it cannot
  break the MAC. Recorded as
  OPEN and untested since 2026-05-09.
- **`freqDelta` at `0x47177` / `0x47179` and the M0205 training table.** Outside every MAC range.
  `freqDelta` is ±1000 on the 8 GB image and 0 on the 10 GB image, so writing ±1000 into the 10 GB
  image and checking whether core offsetting appears is a clean natural experiment.
- **HULK cert injection at `0xFE504`.** The slot is 1113 bytes, all-zero on stock, and because the
  170HX license region is at `0xFE000` it falls inside the 1020 KB window nvflash writes. Whether
  the license region is exempt from the FwSecLic check has never been tested. Three production certs
  from the Lapsus leak (`HULK_9970`, `HULK_12231`, `HULK_12549`) carry `STRICT_ID_MATCH=NO` and
  target `FUSE_FEATURE_OVERRIDE 0x823800`, and were described as "ready to test as-is". **No test
  result is reported anywhere.**

### 3.8 A driverless unlock that hands off to a stock, signed driver

**Wanted:** an unlock surviving Secure Boot and Windows code integrity.
**Why it keeps failing:** the stock driver's booter rejects post-fire SEC2 state with the two-load
"mutex horns" (`0x31` mutex held, `0x62` WPR2 up, `0x29` from `0x1180f8` because the `mutexfree`
terminator leaves the top nibble at `0xf`). This fails even at 10 GB with consistent geometry,
proving it is the SEC2 handoff state that the fire perturbs, not the geometry and not the write
count. **The shipping unlocker sidestepped it by staging geometry from inside a patched driver.**
Two candidate root fixes remain: preserve the booter success path so SEC2 primes its RTOS and sets
DONE itself, or restore the AON `SECURE_SCRATCH` PLM and priv state, which today only a power-domain
reset achieves.

### 3.9 Porting to other silicon

- **CMP 90HX (GA102).** Host-side stage 1 is complete and the targets are known:
  `FEAT_OVR_PLM 0x823804 = 0xffffff8f` (self-sealed via `FUSE_QUADRO_WR_SEC = 1`),
  `FEAT_OVR_SM_SPD 0x82381c = 0x16334012` (target `0x88888888`), `FUSE_EN_SW_OVERRIDE = 1`,
  `FUSE_FEAT_OVR_DIS = 0`. **The blocker is that the SEC2 HS Booter IMEM is encrypted in the GA102
  VBIOS**, so static analysis yields nothing. One analysis concluded the GA102 booter has no
  overflow point; the counter-proposal is not to use the GA10x booter at all but to load the
  **TU10x** booter on the GA10x part, supported by a report that TU10x and GA10x debug booters use
  the same AES-128 secret and one report of the TU10x `booter_load` loading on a 90HX with PLM
  writes succeeding (self-qualified as "1 positive test isn't enough"). Only compute is worth
  unlocking: the 90HX physically carries 10 GB of GDDR6X.
- **CMP 50HX (TU102).** Different memory access-control register set entirely: "there is no single
  PLM there but there seem to be several other masks for the interesting registers."
- **PG199 / DRIVE A100 (`10de:20bb`).** Boot fails with `Booter failed 0x54` after
  `kflcnResetIntoRiscv 0x0`, even though the PLMs open and the CFG1/LMR writes land. Nobody knows
  what `0x54` means, and the status codes `0x31`/`0x47`/`0x15`/`0x29`/`0x88`/`0x96` were all pinned
  by locating their write sites in the booter disassembly, which is already in hand. The branch
  named `PG199` contains **no PG199 support**.

### 3.10 The NVGI pre-firmware write primitive

NVGI records in the SPI ROM are executed by the PBUS/XVE init-from-ROM sequencer *before any
firmware runs*, i.e. before FWSEC raises the PLMs, whereas everything tried on the fuse bank so far
has been at HS runtime after lockdown. Entirely untested.
**Caveats stated by its own proposer:** NVGI is single-copy with no `+0x60000` fallback, so a bad
write is unrecoverable without a 1.8 V SPI programmer; it is "a fancy footgun that could cause big
issues".
**Next step:** trace the record format read-only from a ROM dump before contemplating any write.

### 3.11 The 38 missing SMs

**Wanted:** 70 SM to something nearer 108.
**Tried and failed:** `FUSE_CTRL_OPT_TPC_GPC` writes (remove-only), HS writes to `OPT_GPC_DISABLE`,
`STATUS_OPT_GPC`, `OPT_TPC_GPC2` and `DIS_SW_OVR` (all bounce), and forcing `gpcMask` three ways
(`0x408970` re-asserts to `0xdc`; `cuInit` segfaults).
**Untried candidates named in-channel:** a GSP-RPC/RM control path via the static
floorsweeping-masks queries (`0x2080122a` / `0x2080122b`), a GR-shadow write, or porting the ROP to
PMU / GSP / FECS / GPCCS.
**Sobering context:** on cards where `OPT_GPC_DEFECTIVE = 0` the disabled GPCs are physically good,
so there is something to win, but every write path found so far is latched. One card was found
CTRL_OPT-swept to 56 SM instead of the fuse-floor 70, with 6 SM clawed back to reach 62; that is a
*rare* case, and every other surveyed card is already at its fuse floor.

---

## Tier 4: measurement disputes needing one controlled session

These are not unknowns so much as numbers that were never taken under one methodology. Each needs
one tester, one card, one session, with the tool, the clock and the flop-counting convention stated.

> [!TIP]
> **Closed: the unlocked FP64 spread**
>
> FP64 used to sit in this table as an 11.48-to-12.91 versus 6.3 TFLOPS dispute. It is resolved,
> and it was never a counting error. A single clpeak dump on 2026-07-15 printed both figures in
> one run on one card: **FP64 non-tensor 6.31 TFLOPS** (`double : 6308.65`, exactly half the same
> run's `float : 12565.14`, i.e. the architectural 1:2 rate) and **FP64 tensor 11.96 TFLOPS**
> (`wmma_fp64`, the 8x8x4 DMMA path). The high cluster is the tensor path. See
> [compute-throttle.md](../unlock/compute-throttle.md).

| Dispute | The spread | What settles it |
|---|---|---|
| HBM bandwidth | **1305.86 to 1600 GB/s** across sources. Theoretical peak is 1555.2 GB/s (1215 MHz DDR × 5120-bit) | Run one tool on an 8 GB-to-64 GB card and a 10 GB-to-40 GB card in the same session |
| INT8 / INT4 | 44.1, 50.5, 276, ~280, ~300, 320.2, 335.6 TOPS have all been reported. INT4 measured *below* INT8, where Ampere should be ~2x | One INT8 and one INT4 CUTLASS GEMM at a fixed shape on a card with recorded clock and unlock state, published with the kernel |
| TF32 | 79 to 94 TFLOPS across seven tools. Whether TF32 is throttled at all on a stock card is disputed | Torch GEMM, `gemm_probe.cu` and clpeak back to back in one session, plus one TF32 run on a card confirmed locked by `0x00823818 != 0` |
| L2 cache | `deviceQuery` prints 32768 KB; the spec database says 8 MB; a microbenchmark independently found 32 MB | A published pointer-chase latency curve showing where the working set falls off |
| Shader count | `deviceQuery` prints 8960 SPs and 25.27 TFLOPS; the arithmetic favours 4480 and 12.63 TFLOPS (the tool is applying the cc 8.6 figure of 128 FP32 lanes per SM instead of GA100's 64) | Already effectively settled by arithmetic; recorded because no source resolves it experimentally |
| Memory clock | 304, 364, 432, 864, 1215, 1458, 1728 and 1890 MHz all appear. Different tools use different clock domains and multipliers, and nobody states the conversion | One card, one session: `nvidia-smi -q -d CLOCK`, mixbench and the bandwidth sweep captured together, with the VBIOS version |
| `clocks.max.sm = 1935 MHz` | A reported field, not an operating clock. Every sustained measurement sits at 1410 MHz (1470 MHz at `-pl 300`), and the VBIOS table maximum is 1695 MHz | One re-check on a second card |
| Single-card 27B decode | 97, 90, 75 and 58.5 t/s on vLLM, plus 36.87 t/s for llama.cpp Q4_K_M, differing in backend, quantisation, MTP state, context length and vLLM version. Each figure is a single report | Run one published repo configuration verbatim and post the flags |
| PCIe sensitivity | Width barely matters (+2% single-card) versus generation mattering a lot (+44% on 3-card pipeline-parallel prefill) | A 2×2 matrix of {Gen1, Gen2} × {x4, x16} on one rig with one model |
| Idle power | 27 W to 46 W. Known confounders: card variant, die temperature (leakage feedback) and whether a CUDA context holds a model resident (~+12 W) | One card, one host, idle draw logged at three controlled die temperatures with and without a resident context |
| Long-term stability | Longest verified clean runs are 2-hour stress tests across four cards, a 10-minute cuBLAS run, and second-hand reports of 1-2 day LLM runs. A 24-hour and a 48-hour run were both started and never reported | Publish those results |

---

## Physical questions needing an X-ray, a decap, or a caliper

- **How many HBM stacks does the package carry, and from which vendor?** A delidded 8 GB card
  visibly showed six stacks with four of twelve FBPs swept off; a die-shot source claims two of the
  six are dummies; a third reading is simply four stacks. All three predict the same measured
  4096-bit bus, so bus width discriminates nothing. A circulating guide instead claims five stacks
  of 16 GB, which would imply a 5120-bit bus and does not fit the measurement. The strong working
  assumption (SK hynix on 8 GB, Samsung on 10 GB) has never been verified by package markings, X-ray, or a vendor ID readout. **Note the
  geometry does not depend on it:** the CFG1 profile is the same either way.
- **Where are R240/R241 (DEVID_SEL) on the board?** Wanted because the device ID is what the driver
  keys geometry off, and `OPT_DEVID_SW_OVERRIDE_DIS = 1` closes every software route. The search
  heuristic (a resistor with an empty pad next to it, in the 200-series designator region, on the
  PCB side carrying sub-500 designators) is sound and untried at scale. Two researchers disagree
  about whether the candidate area is near R986-R1005 or on the opposite side.
- **Is the thermal pad 1.5 mm or 2 mm?** One measurement from an owner with the card open, one
  "I think" from someone else. A caliper settles it.
- **Is the unpopulated 4-pin shroud pad actually a fan header?** One report of 12 V and ground
  present, no tach or PWM; a skeptic in the same thread said it does not look like a fan connector.
  No photo-confirmed pinout, no working fan install.

## Unanswerable from the record

- **Total production volume.** Estimates span 10k to hundreds of thousands. The published estimate
  derives unit counts by dividing FY2022 CMP revenue by an assumed average selling price with no
  per-model sales mix. Per-model shipment data does not exist publicly. The honest answer is that
  no reliable figure exists.
- **The 2026-07-05 message treating Gen2 as already unlocked**, three weeks before the reproduced
  result and in direct conflict with a 2026-07-07 message saying "We still need something for
  PCIe 2.0". Either an earlier independent result that never propagated, or a mis-attributed
  timestamp. Only the original message metadata can settle it. The technical content of that
  message (Gen3 is fuse-gated) is consistent with everything else and can be relied on; the date
  cannot.
- **Whether the shipping tool is clean by the clean room's own rules.** Every constant in the
  shipping payload is independently derivable from dated public material that predates the leaked
  artefact, and the clean room had shipped an equivalent chain days earlier. It is also true that
  the shipping code appeared 70 minutes after the leaked artefact, with one hostile change removed
  and one device-ID branch added. Derivability in principle is not derivation in fact. Nothing in
  the record settles it; see [clean room and provenance](../history/clean-room-and-provenance.md).
- **Whether the leaked package was leaked or independently reimplemented.** Both positions were held
  by people close to the work, and the artefact is functionally equivalent either way.

## Related pages

[Status board](status-board.md) ·
[The 80 GB problem](80gb.md) ·
[Gen3 and Gen4](pcie-gen3-gen4.md) ·
[NVLink](nvlink.md) ·
[ECC](ecc.md) ·
[P2P](p2p.md) ·
[Dead ends](../history/dead-ends.md) ·
[Register reference](../unlock/register-reference.md) ·
[Glossary](../start/glossary.md)
