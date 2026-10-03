# Fuses and OTP

## What this page covers

Every documented one-time-programmable (OTP) fuse and fuse-shadow register on the CMP 170HX,
what each one gates, what it reads on this card, what the same register reads on an A100 and on
consumer Ampere, and whether it can be overridden. The data comes from a cross-card BAR0 survey
run in May 2026 across 15 Ampere cards, re-checked here register by register against the
published table.

Two results dominate everything else on this page:

1. **The 170HX is a fully-configured GA100 that has been fused down, not a smaller die.** Every
   scalability register reports the full A100 topology, and every restriction (arithmetic
   throughput, PCIe generation, NVLink, ECC, memory capacity) is an OTP fuse value, not missing
   silicon. See [GA100 silicon](ga100-silicon.md).
2. **The one fuse that would have made all of it permanent is unblown.**
   `FUSE_FEAT_OVR_DIS` at `0x008203F0` reads `0x00000000` on every card ever probed. That is the
   master kill for the feature-override block, and because it is clear, the feature-override
   registers at `0x8238xx` remain live. The entire compute unlock rests on this one zero.

A third result is the one that aged: the survey concluded in bold that the
**host-level register write approach was "CONFIRMED DEAD" for compute unlock**. That conclusion
was correct for what it tested and was overturned six to eight weeks later, not by finding a
host write path but by changing the privilege level mask first from inside a Falcon. See
[How the survey's conclusion was overturned](#how-the-surveys-conclusion-was-overturned).

---

## Reading this page

### Register families

GA100 exposes fuse state at three layers. The probe reads all three, and comparing them is how
you tell a fuse from a software override.

| Prefix | Meaning | Writable? |
|---|---|---|
| `FUSE_*` / `OPT_*` | Raw OTP fuse readout, sensed at boot into a shadow register | No. Burning fuses requires the fuse-programming path, which is closed on production parts |
| `CTRL_OPT_*` | Effective control word: the merge of fuse value, critical-path mask and defective mask. Documented as writable | Only if `FUSE_EN_SW_OVERRIDE` is set. It is not, on this card |
| `STATUS_OPT_*` | Final effective state after all merging. Read-only | Never |

The resolution order, as the register naming implies and as the measured values confirm:

```text
OTP fuse array
   |  (sensed at power-on, FUSECTRL.SENSE_DONE)
   v
FUSE_* shadow  ----+
                   |
CTRL_OPT_* override|--> merge --> STATUS_OPT_*  --> consumed by DevInit / GSP-RM / driver
   (gated by       |
    FUSE_EN_SW_OVERRIDE = 0 on this card)
                   |
DEFECTIVE masks ---+
```

On the CMP 170HX every `CTRL_OPT_*` register reads `0x00000000` and every `STATUS_OPT_*`
register is **bit-for-bit identical to its raw `FUSE_*` counterpart**. Verified here for all four
floorsweep pairs on both physical units. Nothing has been overridden; the restrictions arrive
straight out of the fuse array.

### PRI sentinel values

A read that returns a value beginning `0xBADF` is not a register value. GA100 uses several
distinct sentinels and they mean different things:

| Sentinel | Meaning | Where it shows up in this survey |
|---|---|---|
| `0xBADF5040` | Privilege violation or undecoded space in a priv-gated block | `FECS_FEAT_OVERRIDE` `0x00409664` and `FECS_FEAT_READOUT_1` `0x00409668` on **every** card; `CTRL_OPT_FBPA`, `FUSE_DIS_PROGRAM`, `FUSE_BYPASS_STATUS`, `FUSE_FBPA_DISABLE` and the whole IEEE 1500 block on GA10x parts |
| `0xBADF1100` | Target does not exist / not decoded | `PMC_BOOT_42` `0x0000A800` on all Ampere; `FUSE_OPT_FBIO_OLD` `0x00021C14` on GA100; per-FBPA reads beyond the part's FBPA count |
| `0xBADF1201` | Privilege error, or an engine that has not been ungated | All five SKED registers and `SM_ISSUE_RATE_MODIFIER` on a stock 170HX with **no driver loaded**; this survey read them successfully because `probe.sh` runs `nvidia-smi` and brings the driver up before it mmaps BAR0 |
| `0xBADF20xx` | Floorswept partition, low byte encodes the unit | Per-FBPA `CSTATUS_RAMAMOUNT` and `CFG0` for disabled FBPAs |
| `BAR0` (literal) | Register lies outside the card's BAR0 aperture | Every row at or above offset `0x40000` on the A16, whose BAR0 is only 256 KB |

Any tool that buckets all `BADF*` reads as "unreadable" throws away the difference between a
floorswept unit, an undecoded address and a privilege denial.

### The cohort

| Column | Part | Basis |
|---|---|---|
| 170HX A / 170HX B | 2 × CMP 170HX 10 GB (`10de:2082`) | Physical hardware, 2026-05-05 and 2026-05-07 |
| A100 SXM4 40G, A100 PCIe 40G, A100 PCIe 80G, A10, A16, A5000, A6000, RTX 3080, RTX 3080 Ti, RTX 3090, RTX 3090 Ti | 11 parts | Cloud rentals |
| Drive | 2 × DRIVE A100 32 GB (PG199, `GA100-550F-A1`, `10de:20bb`) | Physical hardware, 2026-05-31, merged into one column |
| ES | An Ampere engineering sample | **No data.** The column exists in the source table and is empty in all 121 rows |

> [!WARNING]
> **Two limits on the cohort you must know before quoting it**
>
> **Both physical 170HX units in this survey are 10 GB (`0x2082`) cards.** No 8 GB (`0x20C2`)
> card was probed. Where the 8 GB SKU differs (notably `NV_PTOP_FS4`, see
> [Chip ID and PTOP](#chip-id-and-ptop)) this table cannot settle it.
>
> **The `ES` column carries no values at all.** Documents that attribute a specific reading to
> "the ES part" have miscounted the columns by one; the ES cell is empty in every row of the
> published table. This was re-verified programmatically for this page.

### What was actually read

`probe.sh` mmaps `/sys/bus/pci/devices/<bdf>/resource0` read-only and issues 32-bit loads. Per
card it reads **120 named registers** plus **24 per-FBPA `CSTATUS_RAMAMOUNT`** plus **24 per-FBPA
`CFG0`** = 168 reads. The published table tabulates **118 unique registers in 121 rows** (three
NVLink registers appear in two sections each) and silently drops two registers the script reads:
`FUSE_FB_FALCON_PRI_DIS` (`0x00820670`) and `PTOP_SCAL_FBPA_PER_FBP` (`0x00022458`). Their 170HX
values are therefore **unknown**, which is a real gap: the first of those decides whether a Falcon
can touch FB PRI registers at all.

---

## Cross-unit comparison: which fuses are product-line and which are per-die

Comparing the two physical 170HX units answers a question that no single dump can: which of these
values is a *product decision* and which is *this particular die*.

Re-counted for this page: of the 121 named-register rows, **108 are byte-identical between the two
units and 13 differ**. The 13 are:

| Register | Address | 170HX A | 170HX B | Class |
|---|---|---|---|---|
| `FEAT_OVR_QUADRO` | `0x00823808` | `0x00000182` | `0x00000181` | runtime/per-die |
| `FEAT_OVR_SM_SPD` | `0x0082381C` | `0x51261070` | `0x10206152` | runtime state, not fuse state |
| `FEAT_OVR_SM_SPD_1` | `0x00823820` | `0x00000002` | `0x00000006` | runtime state, not fuse state |
| `FUSE_GPC_DISABLE` | `0x00820350` | `0x000000d0` | `0x00000023` | per-die binning |
| `FUSE_FBP_DISABLE` | `0x00820364` | `0x00000180` | `0x00000009` | per-die binning |
| `FUSE_FBPA_DISABLE` | `0x00820368` | `0x0003c000` | `0x000000c3` | per-die binning |
| `FUSE_FBIO_DISABLE` | `0x0082036C` | `0x0003c000` | `0x000000c3` | per-die binning |
| `STATUS_FBPA` | `0x00820C18` | `0x0003c000` | `0x000000c3` | mirror of the above |
| `STATUS_FBP` | `0x00820D38` | `0x00000180` | `0x00000009` | mirror |
| `STATUS_OPT_FBIO` | `0x00820C14` | `0x0003c000` | `0x000000c3` | mirror |
| `STATUS_OPT_GPC` | `0x00820C1C` | `0x000000d0` | `0x00000023` | mirror |
| `I1500_DATA` | `0x009A3CBC` | `0xde79ffc1` | `0xc631ffc1` | HBM silicon ID |
| `I1500_SHADOW_WDR` | `0x009A3CC4` | `0xbcf3ff83` | `0x8c63ff83` | HBM silicon ID |

Every restriction fuse matches exactly across the two units: all nine speed-select fuses, both
PCIe fuses, `MAGIC_D`, the NVLink fuses, `FBPA_CFG1_BROADCAST`, `FUSE_SKU_ID`, both device-ID
fuses. **Restriction fuses are product-line; floorsweep fuses are per-die.** That is the single
most useful structural finding in the survey, and it is why one card's floorsweep mask tells you
nothing about another card's, while one card's `FUSE_SS_FFMA` tells you every 170HX's.

The published summary says "107 of 120 registers identical". The differing count of 13 is right;
the ratio is off by one on each side because of the three duplicated NVLink rows.

---

## Fuse controller and the master switches

| Register | Address | 170HX A | 170HX B | A100 (all 3) | A10 / A5000 / A6000 | RTX 30-series | Drive |
|---|---|---|---|---|---|---|---|
| `FUSECTRL` | `0x00820000` | `0xe0040000` | `0xe0040000` | `0xe0040000` | `0xe0040000` | `0xe0040000` | `0xe0040000` |
| `FUSE_EN_SW_OVERRIDE` | `0x00820040` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000001` | `0x00000000` |
| `FUSE_EN_PROGRAM` | `0x00820078` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_DIS_PROGRAM` | `0x0082007C` | `0x00000000` | `0x00000000` | `0x00000000` | `0xbadf5040` | `0xbadf5040` | `0x00000000` |
| `FUSE_BYPASS_STATUS` | `0x00820080` | `0x00000000` | `0x00000000` | `0x00000000` | `0xbadf5040` | `0xbadf5040` | `0x00000000` |
| `FUSE_DIS_SW_OVR` | `0x00820084` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_QUADRO_WR_SEC` | `0x0082038C` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_FEAT_OVR_DIS` | `0x008203F0` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `FUSE_DEVID_SW_OVR_DIS` | `0x00820584` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_INTERNAL_SKU` | `0x008203F4` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |

What each one means for you:

- **`FUSECTRL` `0xe0040000`** decodes as `CMD[1:0]` = 0 (idle), `STATE[20:16]` = 4,
  `SENSE_DONE[30]` = 1, plus bits 29 and 31 set (not named in the probe catalogue). Identical on
  all 14 cards with data. The fuse array has been sensed and the controller is idle.
- **`FUSE_EN_SW_OVERRIDE` = `0x00000000`** is the reason the `CTRL_OPT_*` path is a dead end on
  this card. This is the cleanest architecture-versus-tier split in the whole survey: **zero on
  every GA100 datacentre part (both 170HX units, all three A100 SKUs, the Drive A100), one on
  every consumer and professional GA10x part.** The 170HX is not specially locked relative to a
  normal A100 here; the whole GA100 datacentre line ships this way. The corollary is that the
  25-entry `NV_FUSE_CTRL_OPT_*` table sitting at offset `0x47341` in the unsigned FwSec VBIOS
  tail is inert on this hardware, and reads all zero on 13 probed GA100 cards. See
  [VBIOS](vbios.md).
- **`FUSE_DIS_SW_OVR` = `0x00000001`** on all cards. Best current reading: it locks
  `EN_SW_OVERRIDE` at whatever value it already holds. That interpretation has never been tested
  with a write probe and remains an inference.
- **`FUSE_QUADRO_WR_SEC` = `0x00000001`** seals the privilege level mask at `0x00823804`. This is
  the self-sealing part of the chain: the fuse protects the PLM, the PLM protects the override
  registers.
- **`FUSE_FEAT_OVR_DIS` = `0x00000000`** is the load-bearing zero. Blown, it would permanently
  lock every feature override and there would be no compute unlock and no wiki.
- **`FUSE_DEVID_SW_OVR_DIS` = `0x00000001`** on every card. Software cannot change the presented
  PCI device ID. Every "flash it as an A100" plan dies here, at the fuse level, before firmware
  signing even becomes relevant.

> [!CAUTION]
> **Do not write the fuse-programming registers**
>
> `FUSE_EN_PROGRAM` reads `0x00000001` on every card and the probe catalogue annotates it
> "do not use". `FUSECTRL`, `FUSE_EN_PROGRAM`, `FUSE_DIS_PROGRAM` and the fuse data registers
> drive an OTP array. A successful write is permanent and unrecoverable, and a partial burn can
> leave the card unable to sense a valid configuration at power-on. Nothing in this project has
> ever needed a fuse write, and nothing in the shipping unlocker attempts one.

---

## SM speed select OTP fuses

These are the arithmetic throttle. Each is a 3-bit field where `0` is full rate and `5` is
divide-by-32, except `FUSE_SS_DP`, which is a single bit where `1` means reduced.

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | A10 | A5000 | A6000 | RTX 3080 | RTX 3080 Ti | RTX 3090 | RTX 3090 Ti | Drive |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `FUSE_SS_DP` | `0x00820224` | `0x00000001` | `0x00000001` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_FFMA` | `0x0082059C` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_FMLA16` | `0x008207D4` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_FMLA32` | `0x008207D8` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0` |
| `FUSE_SS_IMLA0` | `0x008207DC` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_IMLA1` | `0x008207E0` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_IMLA2` | `0x008207E4` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_IMLA3` | `0x008207E8` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_SS_IMLA4` | `0x008207EC` | `0x00000005` | `0x00000005` | `0` | `0` | `0` | `0` | `0` | `0` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0` |
| `FUSE_SS_PLM` | `0x008200FC` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` | `0xffffffff` |

The pattern across the whole Ampere line:

- **CMP**: eight fuses at `0x5` (divide-by-32) plus DP at `0x1`. Maximum throttle on every
  arithmetic unit the fuse block can reach.
- **Consumer GeForce** (RTX 3080, 3080 Ti, 3090, 3090 Ti): `FMLA32` and `IMLA4` at `0x1`, all
  others zero. This is the well-known consumer FP32-accumulate tensor restriction, and it is
  fused, not driver-side.
- **Professional and datacentre** (A100 ×3, A10, A5000, A6000, Drive A100): all nine at zero.

> [!NOTE]
> **Correction to the survey's own headline**
>
> The published key findings say "ALL 9 speed select fuses at `0x5`". Eight read `0x5`;
> `FUSE_SS_DP` reads `0x1` because it is a 1-bit fuse, and `0x1` is its maximum. The
> substance ("every arithmetic unit is maximally throttled, CMP-exclusive") is correct.

**Are these overridable? No, but they are bypassable.** Nothing can un-blow them. The
feature-override block at `0x8238xx` sits downstream of the fuse readout and takes precedence,
and that is what the shipping unlock uses. `FUSE_SS_PLM` (`0x008200FC`, also called `OPT_PLM` in
the branch source) is the shared privilege level mask for the speed-select fuses; it already
reads `0xffffffff` here, and **the shipping unlocker never writes it**. Full mechanism on
[Compute throttle](../unlock/compute-throttle.md).

> [!NOTE]
> **Open problem: what does `0x008200FC` actually read on a cold card?**
>
> This survey reads `0xffffffff` on all 14 cards. A separate 2026-07-09 read on a 170HX
> recorded `0x000003FF`. A nine-PLM branch variant that tries to write it logs
> `status=0xffff` with no readback captured, and `0xffff` is the Booter's status for every run
> regardless of outcome. Whether the register is writable, and what it reads before any tooling
> has touched it, is unresolved.

---

## Feature override chain

The block at `0x00823800` to `0x0082382C`. This is where the unlock happens, so it is worth
having every value.

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | A10 | A5000 | A6000 | RTX 3080 | RTX 3080 Ti | RTX 3090 | RTX 3090 Ti | Drive |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `FEAT_OVR_ECC_PLM` | `0x00823800` | `0xffffff8f` | `0xffffff8f` | `0x0000abcf` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` |
| `FEAT_OVR_PLM` | `0x00823804` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` | `0xffffff8f` |
| `FEAT_OVR_QUADRO` | `0x00823808` | `0x00000182` | `0x00000181` | `0x00000080` | `0x00000180` | `0x01100080` | `0x01000083` | `0x01000081` | `0x01100180` | `0x00000080` | `0x01100082` | `0x00000383` | `0x01000383` | `0x00000380` |
| `FEAT_OVR_ECC` | `0x0082380C` | `0x00888888` | `0x00888888` | `0x00000100` | `0x00001111` | `0x00110111` | `0x00800888` | `0x00888888` | `0x00888888` | `0x00888888` | `0x00888888` | `0x00888888` | `0x00888888` | `0x00101110` |
| `FEAT_OVR_ECC_1` | `0x00823810` | `0x002aaaaa` | `0x002aaaaa` | `0x00100400` | `0x00104015` | `0x00104104` | `0x005004aa` | `0x002aaaaa` | `0x002aaaaa` | `0x002aaaaa` | `0x002aaaaa` | `0x002aaaaa` | `0x00aaaaaa` | `0x00040000` |
| `FEAT_READOUT_0` | `0x00823814` | `0x00000233` | `0x00000233` | `0xef8ff100` | `0xef8ff100` | `0xef8ff100` | `0xee018300` | `0x00000300` | `0x00000300` | `0x00000233` | `0x00000233` | `0x00000233` | `0x00000233` | `0xef8ff100` |
| `FEAT_READOUT_1` | `0x00823818` | `0x016db6ed` | `0x016db6ed` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00400080` | `0x00400080` | `0x00400080` | `0x00400080` | `0x00000000` |
| `FEAT_OVR_SM_SPD` | `0x0082381C` | `0x51261070` | `0x10206152` | `0x10413004` | `0x14604062` | `0x72020072` | `0x11303071` | `0x63573073` | `0x14170072` | `0x03676064` | `0x10551033` | `0x06740057` | `0x30403100` | `0x25045144` |
| `FEAT_OVR_SM_SPD_1` | `0x00823820` | `0x00000002` | `0x00000006` | `0x00000004` | `0x00000001` | `0x00000001` | `0x00000004` | `0x00000003` | `0x00000006` | `0x00000006` | `0x00000007` | `0x00000007` | `0x00000004` | `0x00000003` |
| `FEAT_OVR_ROW_REMAP` | `0x00823824` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000003` | `0x00000000` |
| `FEAT_READOUT_2` | `0x00823828` | `0x00000000` | `0x00000000` | `0x00000007` | `0x00000007` | `0x00000007` | `0x0000000b` | `0x0000000b` | `0x0000000b` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000007` |
| `FEAT_OVR_ECC_2` | `0x0082382C` | `0x0000000a` | `0x0000000a` | `0x00000001` | `0x00000000` | `0x00000000` | `0x00000009` | `0x00000008` | `0x00000008` | `0x00000008` | `0x00000008` | `0x00000008` | `0x0000000a` | `0x00000000` |

Notes that matter:

- **`FEAT_OVR_PLM` `0x00823804` = `0xffffff8f` is universal.** Level-3 access only. It is
  identical on an A100 with no restrictions and on a 170HX, so this PLM is not the restriction,
  it is just the lock on the toolbox. Opening it to `0xffffffff` is the first of the two things
  the compute unlock does. It is an always-on-island register and **survives FLR**, which is
  exactly why compute shipped before memory. See
  [Privilege level masks](../unlock/privilege-level-masks.md).
- **`FEAT_OVR_ECC_PLM` `0x00823800` is a different register** and is easy to confuse with the
  one above. Note the one outlier in the entire cohort: the A100 SXM4 40 GB reads `0x0000abcf`
  where all 13 other cards read `0xffffff8f`. Unexplained.
- **`FEAT_READOUT_1` `0x00823818` is the effective throttle readout and the best unlock test.**
  Read-only. `0x016db6ed` on both 170HX units, `0x00000000` on every part whose speed-select
  fuses are all zero, `0x00400080` on all four consumer parts whose `FMLA32`/`IMLA4` fuses read
  `0x1`. After a successful compute unlock a 170HX reads `0x00000000` here. The internal field
  layout is not documented anywhere in the corpus, but the three-way correlation with fuse state
  is exact, and `0x00823818 == 0` is the cleanest available "is this card unthrottled" test.
- **`FEAT_OVR_SM_SPD` / `_1` read-backs are runtime state, not fuse state.** They differ between
  the two 170HX units in this very survey (`0x51261070` vs `0x10206152`), and two archived dumps
  of the same A100 80 GB device ID disagree with each other. Do not use them as a per-part
  reference target; use `0x00823818`. The unlock writes `0x88888888` and `0x00000008` here.
- **`FEAT_OVR_QUADRO` `0x00823808`** differs between the two units and also differs across every
  recorded 170HX state: `0x00100183` on a stock card in one dump, `0x00000081` after unlock,
  `0x01000282` on a genuine A100 80 GB. Something in the unlock or the driver version touches the
  Quadro-versus-consumer classification word. Nobody has chased it.
- Undecoded space in this block returns `0xBADF5040` from `0x823830` to `0x823FFC` on a stock
  170HX, which is how the block's extent was established. `0x00823B00`, the row-remapper PLM,
  reads `0xffffff8f` and lies outside the tabulated range.

---

## The throttle red herrings

Two registers were pursued hard as the throttle and are not.

| Register | Address | 170HX A | 170HX B | Everything else with data |
|---|---|---|---|---|
| `SM_ISSUE_RATE_MODIFIER` | `0x00504204` | `0x00000005` | `0x00000005` | `0x00000005` on all 12 other cards, including every A100 |
| `FECS_FEAT_OVERRIDE` | `0x00409664` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` on all 12 |
| `FECS_FEAT_READOUT_1` | `0x00409668` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` on all 12 |

`SM_ISSUE_RATE_MODIFIER` reads `0x00000005` on an unthrottled A100 as well as on the 170HX, so it
cannot be the differentiator. It is host-writable at the right privilege level and zeroing it
produces no measurable performance change. The shipping unlocker never touches it. A GA104
RTX 3070 control card reads `0x7` here, further confirming the value is architectural rather
than a restriction. Note also that this register reads `0xbadf1201` on a stock 170HX with **no
driver loaded**, and reads cleanly once the driver has brought the engines up, which is the state
`probe.sh` leaves the card in before it takes its readings.

The two `FECS_*` registers return the privilege-violation sentinel on **every** Ampere card
probed, so they carry no information about the 170HX at all.

---

## PCIe fuses

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | A10 | A5000 | A6000 | RTX 3080 | RTX 3080 Ti | RTX 3090 | RTX 3090 Ti | Drive |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `FUSE_PCIE_GEN23_DIS` | `0x0082057C` | `0x00000001` | `0x00000001` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_PCIE_GEN3_DIS` | `0x00820580` | `0x00000001` | `0x00000001` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_PCIE_MAGIC_D` | `0x00820520` | `0x16680000` | `0x16680000` | `0x00200000` | `0x00200000` | `0x00200000` | `0x01a00000` | `0x01a00000` | `0x01a00000` | `0x10a80000` | `0x10a80000` | `0x10a80000` | `0x10a80000` | `0x00200000` |
| `FUSE_PCIE_DEVIDA` | `0x008204D8` | `0x00002082` | `0x00002082` | `0x000020b1` | `0x000020b1` | `0x000020b5` | `0x00002236` | `0x00002231` | `0x00002230` | `0x00002216` | `0x00002208` | `0x00002204` | `0x00002203` | `0x000020bb` |
| `FUSE_PCIE_DEVIDB` | `0x0082056C` | `0x000020c2` | `0x000020c2` | `0x000020f1` | `0x000020f1` | `0x000020f5` | `0x00002276` | `0x00002271` | `0x00002270` | `0x00002256` | `0x00002248` | `0x00002244` | `0x00002243` | `0x000020fb` |
| `FUSE_PCIE_LANE_DIS` | `0x00820394` | `0x00000000` | `0x00000000` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |

- **Both speed fuses are blown on the 170HX and clear on all 12 comparison parts.** It is one of
  the handful of places where the 170HX stands alone against every A100 *and* the Drive A100; the
  speed-select fuses, `FUSE_SKU_ID`, `FEAT_READOUT_1` and `FBPA_CFG1_BROADCAST` do the same.
- **`FUSE_PCIE_MAGIC_D` `0x16680000` has bit 25 set**, annotated `GEN4_SPEED_DISABLED` and
  referenced to an internal NVIDIA bug number. The A100 and Drive A100 read `0x00200000`, bit 21
  only, with bit 25 clear.
- **The lane fuses are clean.** `FUSE_PCIE_LANE_DIS`, `CTRL_OPT_PCIE_LANE` (`0x0082082C`) and
  `STATUS_OPT_PCIE_LANE` (`0x00820C2C`) all read `0x00000000`. This is independent silicon-side
  confirmation that the x4 link width is a **board** problem (12 of 16 lanes ship with their
  AC-coupling capacitors depopulated) and not a fuse. Speed and width are two separate
  restrictions with two separate fixes and must never be conflated. See
  [PCIe subsystem](pcie-subsystem.md) and [Physical mods](../operations/physical-mods.md).

> [!NOTE]
> **Correction: `DEVIDB` is not 'the 8 GB device ID'**
>
> The survey highlights that both 10 GB units read `FUSE_PCIE_DEVIDB = 0x20C2`, the 8 GB card's
> PCI ID, and reads significance into it. Checking all 11 parts with data,
> **`DEVIDB = DEVIDA + 0x40` on every single one**: `0x2082`→`0x20c2`, `0x20b1`→`0x20f1`,
> `0x20b5`→`0x20f5`, `0x2236`→`0x2276`, `0x2231`→`0x2271`, `0x2230`→`0x2270`,
> `0x2216`→`0x2256`, `0x2208`→`0x2248`, `0x2204`→`0x2244`, `0x2203`→`0x2243`,
> `0x20bb`→`0x20fb`. `DEVIDB` is a mechanical +0x40 alternate ID, and the coincidence that
> `0x2082 + 0x40` lands on the other 170HX SKU's ID carries no SKU information. **Testable
> prediction:** an 8 GB card should read `DEVIDA = 0x20c2` and `DEVIDB = 0x2102`. No 8 GB card
> was probed, so this is unverified.

**Overridable?** `OPT_GEN23` at `0x0082057c` was targeted directly, twice, from a Booter-mediated
high-secure write and the readback stayed `0x00000001` every time. Gen2 was eventually reached
anyway, through the `CYA_0` / `LINK_CONFIG_0` / XP3G / `PRIV_MISC_1` overrides, **without ever
clearing the fuse shadow**. The fuse is not the lever. Whether `FUSE_PCIE_MAGIC_D` is writable has
never been tested and is a five-minute experiment nobody has published. See
[PCIe Gen 2](../unlock/pcie-gen2.md) and [Gen 3 and Gen 4](../frontier/pcie-gen3-gen4.md).

---

## NVLink fuses

Summarised here; the full treatment including the physical connectors and the boot log is on
[NVLink hardware](nvlink-hardware.md).

| Register | Address | 170HX A | 170HX B | A100 (all 3) | A10 / A5000 / A6000 | RTX 3080 / 3080 Ti | RTX 3090 / 3090 Ti | Drive |
|---|---|---|---|---|---|---|---|---|
| `FUSE_NVLINK_DIS` | `0x00820684` | `0x00000007` | `0x00000007` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000000` | `0x00000007` |
| `FUSE_NVLINK_DEFECTIVE` | `0x0082068C` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `FUSE_NVLINK_DIS_CP` | `0x00820688` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `FUSE_NVLINK_MASK_SEC` | `0x00820704` | `0x00000005` | `0x00000005` | `0x00000005` | `0x00000085` | `0x00000085` | `0x00000085` | `0x00000005` |
| `FUSE_NVLINK_PHYS_DMG` | `0x00820BD4` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_NVLIPT_RST_DIS` | `0x00821100` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `CTRL_OPT_NVLINK` | `0x008209B8` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `STATUS_OPT_NVLINK` | `0x00820DB8` | `0x00000007` | `0x00000007` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000000` | `0x00000007` |
| `PTOP_SCAL_NUM_NVLINK` | `0x0002246C` | `0x0000000c` | `0x0000000c` | `0x0000000c` | `0x00000004` | `0x00000004` | `0x00000004` | `0x0000000c` |

Disabled, not damaged, and not unique to the mining SKU: the Drive A100 32 GB reads the identical
`0x7` / `0x7` pair. **Not overridable by any known route.**

---

## ECC and memory capacity fuses

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | A10 | A5000 | A6000 | RTX 3080 | RTX 3080 Ti | RTX 3090 | RTX 3090 Ti | Drive |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `FUSE_ECC_EN` | `0x00820228` | `0x00000000` | `0x00000000` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0` | `0` | `0` | `0` | `0x00000001` |
| `FUSE_HALF_FBPA_EN` | `0x0082049C` | `0x00000000` | `0x00000000` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_FB_CONFIG` | `0x00820328` | `0x00000000` | `0x00000000` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` | `0` |
| `FUSE_MEM_LOCKED` | `0x00820340` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_SPARE_FS` | `0x00820398` | `0x00000000` | `0x00000000` | `0` | `0` | `0` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0` |
| `FUSE_FBPA_MEM_WR_SEC` | `0x00820618` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` | `0x00000001` |
| `FUSE_SKU_ID` | `0x00821060` | `0x00000068` | `0x00000068` | `0x0000005b` | `0x00000054` | `0x0000007b` | `0x00000039` | `0x00000020` | `0x00000038` | `0x0000000f` | `0x0000000b` | `0x0000000d` | `0x0000003c` | `0x0000008e` |

- **`FUSE_ECC_EN` = `0x00000000`.** ECC is fused off. The split is clean: `1` on every
  professional and datacentre part, `0` on every consumer part **and** on the CMP. No override
  register exists, there is no ECC entry in the feature-override block that anyone has found a
  use for, there is no ECC telemetry, and the branch named `ecc` in the unlocker repository
  contains no ECC code at all (its single commit is "Fixed dual geometry support"). See
  [ECC](../frontier/ecc.md).
- **`FUSE_HALF_FBPA_EN` = `0`, and so do `CTRL_HALF_FBPA` and `STATUS_HALF_FBPA`.** There is no
  half-capacity fusing on this card. This kills the theory that the 512 MiB reported per FBPA is
  a half-FBPA effect, and with it the proposal that clearing a half-capacity fuse would yield
  96 GB or 128 GB.
- **`FUSE_MEM_LOCKED` = `0x00000001` on every card in the cohort, including cards whose memory
  geometry is demonstrably rewritten at runtime.** The shipping unlock changes `FBPA_CFG1` and the
  MMU local-memory range at runtime with this fuse set. Whatever `OPT_MEMORY_LOCKED_ENABLED`
  gates, it is not the `CFG1` write path once the FBPA privilege level mask is open.
- **`FUSE_FBPA_MEM_WR_SEC` = `1` everywhere**, which is what makes the FBPA memory-config
  registers privilege-locked and is why the unlock has to open `0x009a0148` before writing
  `0x009a0204`.
- **`FUSE_SKU_ID` = `0x68`** on both units and distinct from every other Ampere part recorded. A
  separately probed PG199 board reads `0x69`. `FUSE_INTERNAL_SKU` is zero everywhere, so none of
  the probed parts is flagged internal.
- `FUSE_SPARE_FS` and `STATUS_SPARE_FS` split cleanly GA100 (`0`) versus GA10x (`1`), which reads
  as an architecture-era field rather than a product restriction.

---

## Floorsweep fuses

Raw OTP disable masks. These are the per-die binning fuses, and they are the 13 registers that
differ between the two 170HX units.

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | A10 | A5000 | A6000 | RTX 3080 | RTX 3080 Ti | RTX 3090 | RTX 3090 Ti | Drive |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `FUSE_GPC_DISABLE` | `0x00820350` | `0x000000d0` | `0x00000023` | `0x00000008` | `0x00000008` | `0x00000080` | `0x00000020` | `0x00000008` | `0x00000000` | `0x00000001` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000044` |
| `FUSE_FBP_DISABLE` | `0x00820364` | `0x00000180` | `0x00000009` | `0x00000012` | `0x00000840` | `0x00000840` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000008` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000852` |
| `FUSE_FBPA_DISABLE` | `0x00820368` | `0x0003c000` | `0x000000c3` | `0x0000030c` | `0x00c03000` | `0x00c03000` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` | `0xbadf5040` | `0x00c0330c` |
| `FUSE_FBIO_DISABLE` | `0x0082036C` | `0x0003c000` | `0x000000c3` | `0x0000030c` | `0x00c03000` | `0x00c03000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000008` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00c0330c` |

Decoded for the two 170HX units:

| Unit | GPC mask | GPCs disabled | FBP mask | FBPs disabled | FBPA mask | FBPAs disabled | Active FBPAs |
|---|---|---|---|---|---|---|---|
| 170HX A | `0x000000d0` | 4, 6, 7 | `0x00000180` | 7, 8 | `0x0003c000` | 14, 15, 16, 17 | 20 of 24 |
| 170HX B | `0x00000023` | 0, 1, 5 | `0x00000009` | 0, 3 | `0x000000c3` | 0, 1, 6, 7 | 20 of 24 |

Both units disable **exactly three of eight GPCs and two of twelve FBPs**, but different ones.
The FBP mask and the FBPA mask agree with each other on both units, at two FBPAs per FBP, which is
a useful internal consistency check (FBP 7 owns FBPAs 14 and 15; FBP 8 owns 16 and 17; FBP 0 owns
0 and 1; FBP 3 owns 6 and 7).

Five GPCs at 8 TPCs per GPC and 2 SMs per TPC would allow up to 80 SMs. A live 8 GB card reports
**70** SMs to CUDA, so a further five TPCs must be swept at TPC granularity. `probe.sh` does not
read any TPC disable register, so the TPC mask on this card is **unknown**. The same arithmetic
sanity-checks against the A100 SXM4 40 GB in the cohort: one GPC disabled leaves 112 SMs
available at GPC granularity against the A100's shipped 108, so two TPCs are swept there too.

Comparison capacities, all confirmed by the arithmetic:

| Part | Active FBPAs | Per-FBPA `CSTATUS` | Capacity |
|---|---|---|---|
| CMP 170HX 10 GB | 20 | `0x200` (512 MiB) | 10240 MiB |
| A100 SXM4 40 GB | 20 | `0x7ff` | 40 GB |
| A100 PCIe 40 GB | 20 | `0x7ff` | 40 GB |
| A100 PCIe 80 GB | 20 | `0xfff` | 80 GB |
| Drive A100 32 GB | 16 (mask `0x00c0330c` disables 2, 3, 8, 9, 12, 13, 22, 23) | `0x7ff` | 32 GB |
| RTX 3080 | 5 (one partition swept, `fbpa03`) | `0x800` | 10 GB |

The 170HX therefore has the **same number of active memory partitions as an A100 40 GB** and each
one reports one quarter of the capacity. That is the whole memory story in one line, and it is
what the unlock reverses. See [Memory subsystem](memory-subsystem.md) and
[Memory geometry](../unlock/memory-geometry.md).

> [!NOTE]
> **The GA10x `0xbadf5040` on `FUSE_FBPA_DISABLE`**
>
> Every GA10x part returns the privilege sentinel for `0x00820368` while returning a real value
> for `0x0082036C` (FBIO). The floorsweep on those parts has to be read from the FBIO mask
> instead. This is an architecture difference in PRI gating, not a measurement failure.

---

## Effective CTRL

Every one of these reads `0x00000000` on both 170HX units, and on every other card except where
noted.

| Register | Address | 170HX | Exceptions in the cohort |
|---|---|---|---|
| `CTRL_HALF_FBPA` | `0x00820800` | `0x00000000` | none |
| `CTRL_FB_CONFIG` | `0x00820834` | `0x00000000` | none |
| `CTRL_OPT_FBIO` | `0x00820814` | `0x00000000` | none |
| `CTRL_OPT_FBPA` | `0x00820818` | `0x00000000` | `0xbadf5040` on all seven GA10x parts |
| `CTRL_OPT_GPC` | `0x0082081C` | `0x00000000` | none |
| `CTRL_OPT_PERLINK` | `0x00820820` | `0x00000000` | none |
| `CTRL_OPT_PCIE_LANE` | `0x0082082C` | `0x00000000` | none |
| `CTRL_OPT_FBP` | `0x00820938` | `0x00000000` | none |
| `CTRL_OPT_NVLINK` | `0x008209B8` | `0x00000000` | none |

An all-zero `CTRL_OPT` block on a card with heavy floorsweeping is the proof that the disables
arrive through the fuse and STATUS path, not through a control override somebody set. It is also
why writing these registers is the most-cited unexplored lever in the project: they look
writable, they are documented as the effective control, and they are empty. The reason nobody
expects it to work is `FUSE_EN_SW_OVERRIDE = 0`.

## Effective STATUS

| Register | Address | 170HX A | 170HX B | Equals which raw fuse? |
|---|---|---|---|---|
| `STATUS_HALF_FBPA` | `0x00820C00` | `0x00000000` | `0x00000000` | `FUSE_HALF_FBPA_EN` |
| `STATUS_FBPA` | `0x00820C18` | `0x0003c000` | `0x000000c3` | `FUSE_FBPA_DISABLE` |
| `STATUS_FBP` | `0x00820D38` | `0x00000180` | `0x00000009` | `FUSE_FBP_DISABLE` |
| `STATUS_FB_CONFIG` | `0x00820C34` | `0x00000000` | `0x00000000` | `FUSE_FB_CONFIG` |
| `STATUS_SPARE_FS` | `0x00820C30` | `0x00000000` | `0x00000000` | `FUSE_SPARE_FS` |
| `STATUS_OPT_FBIO` | `0x00820C14` | `0x0003c000` | `0x000000c3` | `FUSE_FBIO_DISABLE` |
| `STATUS_OPT_GPC` | `0x00820C1C` | `0x000000d0` | `0x00000023` | `FUSE_GPC_DISABLE` |
| `STATUS_OPT_PCIE_LANE` | `0x00820C2C` | `0x00000000` | `0x00000000` | `FUSE_PCIE_LANE_DIS` |
| `STATUS_OPT_NVLINK` | `0x00820DB8` | `0x00000007` | `0x00000007` | `FUSE_NVLINK_DIS` |

Perfect agreement in all nine pairs, on both units.

> [!NOTE]
> **Address correction**
>
> `STATUS_OPT_FBPA` is at `0x00820C18`. `0x00820C14` is `STATUS_OPT_FBIO`. On several probed
> dies both happen to hold the same value, which is exactly why the mislabel survived for
> months. The probe script now carries the correction inline.

---

## Chip ID and PTOP

| Register | Address | 170HX A | 170HX B | A100 (all 3) | A16 | GA10x consumer/pro | Drive |
|---|---|---|---|---|---|---|---|
| `PMC_BOOT_0` | `0x00000000` | `0x170000a1` | `0x170000a1` | `0x170000a1` | `0xb77000a1` | `0xb72000a1` | `0x170000a1` |
| `PMC_BOOT_42` | `0x0000A800` | `0xbadf1100` | `0xbadf1100` | `0xbadf1100` | `0x00000000` | `0xbadf1100` | `0xbadf1100` |
| `FUSE_OPT_FBIO_OLD` | `0x00021C14` | `0xbadf1100` | `0xbadf1100` | `0xbadf1100` | `0x00000000` | `0xbadf1002` | `0xbadf1100` |
| `PTOP_FS4` | `0x0002241C` | `0x00000081` | `0x00000081` | `0x00000081` | `0x00000000` | `0x00000081` | `0x00000081` |
| `PTOP_SCAL_NUM_GPCS` | `0x00022430` | `0x00000008` | `0x00000008` | `0x00000008` | `0x00000000` | `0x00000007` | `0x00000008` |
| `PTOP_SCAL_NUM_TPC_GPC` | `0x00022434` | `0x00000008` | `0x00000008` | `0x00000008` | `0x00000000` | `0x00000006` | `0x00000008` |
| `PTOP_SCAL_NUM_FBPS` | `0x00022438` | `0x0000000c` | `0x0000000c` | `0x0000000c` | `0x00000000` | `0x00000006` | `0x0000000c` |
| `PTOP_SCAL_NUM_FBPAS` | `0x0002243C` | `0x00000018` | `0x00000018` | `0x00000018` | `0x00000000` | `0x00000006` | `0x00000018` |
| `PTOP_SCAL_NUM_LTCS` | `0x00022454` | `0x00000018` | `0x00000018` | `0x00000018` | `0x00000000` | `0x0000000c` | `0x00000018` |
| `PTOP_SCAL_NUM_NVLINK` | `0x0002246C` | `0x0000000c` | `0x0000000c` | `0x0000000c` | `0x00000000` | `0x00000004` | `0x0000000c` |
| `PTOP_FS_STATUS` | `0x00022470` | `0x0000003f` | `0x0000003f` | `0x0000003f` | `0x00000000` | varies (`0x63`, `0xf7`, `0x01`, `0x00`) | `0x0000003f` |

`PMC_BOOT_0 = 0x170000a1` is the single strongest piece of evidence that this is a complete
GA100: `0x170` is the GA100 chip-implementation ID, `a1` is the revision that `lspci` reports,
and the value is byte-identical to all three A100 SKUs. GA10x parts read `0xb7`, a GA104
RTX 3070 control reads `0xb74000a1`, and a CMP 90HX reads `0xb72000a1`, confirming the 90HX is a
completely different die.

Every PTOP scalability register reports the **full** GA100 configuration: 8 GPCs, 8 TPCs per GPC,
12 FBPs, 24 FBPAs, 24 LTCs, 12 NVLinks. These describe what the silicon was built with and are
unaffected by floorsweep or disable fuses, which is precisely why they are the reference against
which the fuse damage is measured.

> [!NOTE]
> **Open problem: `NV_PTOP_FS4` on the 8 GB card**
>
> Bit 0 of `0x0002241C` is `GEN2_PCIE` and bit 7 is `GEN2_PCIE_SPEED`. Both 10 GB units in this
> survey read `0x00000081` (both bits set), matching every A100 and every GA10x part. A
> separate probe reports `0x00000000` on an 8 GB (`0x20C2`) card, and the current reconciliation
> treats it as a per-SKU split: `0x00000000` on 8 GB, `0x00000081` on 10 GB. The `0x00` reading
> is the interesting half and it rests on one probe. What would settle it: a fresh read of
> `0x0002241C` on a known `10de:20c2` and a known `10de:2082` card, each reported alongside
> `lspci -nn` for the same bus address.

---

## SKED

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G / 80G | GA10x | Drive |
|---|---|---|---|---|---|---|---|
| `SKED_UNK54` | `0x00407054` | `0x60000600` | `0x60000600` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `SKED_TRAP` | `0x00407020` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `SKED_TRAP_EN` | `0x00407024` | `0x3ffffffc` | `0x3ffffffc` | `0x3ffffffc` | `0x3dfffffc` | `0xbffffffc` / `0xbdfffffc` | `0x3ffffffc` |
| `SKED_HW_BLK` | `0x00407000` | `0x00000000` | `0x00000000` | `0x00004042` | `0x00004042` | `0x00004042` | `0x00004042` |
| `SKED_PM_UNK10` | `0x00407010` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |

Two 170HX-only readings here, both taken before any graphics-engine init:

- **`SKED_UNK54` = `0x60000600`**, the only non-zero reading in the entire 13-card comparison
  set. It is cleared by the driver at graphics-engine init, so this is pre-init fuse-loaded state
  only. A separate probe on an **unlocked 8 GB** card records `0x600000c0` at the same address,
  which is a different value under different conditions, not a contradiction of this one. It is
  the most-referenced undocumented SKED register in the GSP firmware, with 13 references, and
  nobody has diffed those references against the observed values.
- **`SKED_HW_BLK` = `0x00000000`** on both 170HX units where all 12 comparison parts read
  `0x00004042`. A later probe on an unlocked card reads `0x00004042`, so this too appears to be a
  pre-init state rather than a permanent difference.

Bit 31 of `SKED_TRAP_EN` splits cleanly: clear on all GA100, set on all GA10x. Note that on a
stock 170HX with **no driver loaded**, all five SKED registers return `0xbadf1201`; the values
above were read as root over an mmap of BAR0 after `probe.sh` had brought the driver up, but
before any graphics-engine init.

---

## FBHUB and MMU

| Register | Address | 170HX A | 170HX B | A100 (all 3) | GA10x | Drive |
|---|---|---|---|---|---|---|
| `FBHUB_NUM_ACTIVE_LTCS` | `0x00100800` | `0x00000014` | `0x00000014` | `0x00000014` | `0x0000000a`–`0x0000000c` | `0x00000010` |
| `FBHUB_MEM_PART_BOT` | `0x00100B88` | `0x00000000` | `0x00000000` | `0x00000204` / `0x00000204` / `0x00000404` | `0x00000000` | `0x00000202` |
| `FBHUB_MEM_PART_MID` | `0x00100B8C` | `0x00000000` | `0x00000000` | `0x0001f83d` / `0x0001f986` / `0x0003f664` | `0x00000000` | `0x0001f8bf` |
| `FBHUB_MEM_PART_BCFG0` | `0x00100B90` | `0x00000603` | `0x00000603` | `0x00000603` | `0x00000603` | `0x00000603` |
| `FBHUB_HSHUB_SYS_CFG` | `0x00100B98` | `0x00000003` | `0x00000003` | `0x00000003` | `0x00000001` | `0x00000003` |
| `MMU_NUM_ACTIVE_LTCS` | `0x00100EC0` | `0x05001414` | `0x05001414` | `0x05001414` | `0x4420130c` etc. | `0x04001410` |

- `FBHUB_NUM_ACTIVE_LTCS = 0x14` (20) on the 170HX and on every A100, `0x10` (16) on the Drive
  A100 with its 16 active FBPAs. LTC count tracks FBPA count one for one.
- `MMU_NUM_ACTIVE_LTCS` is read-only; its `[4:0]` field carries the LTC count and matches
  (`0x14` = 20 on the 170HX, `0x10` = 16 on the Drive).
- `FBHUB_HSHUB_SYS_CFG` (`SYSMEM_HSHUB_CONNECTION_CFG`, PCIe routing, init `0x3` = BOTH) is `3`
  on every GA100 and `1` on every GA10x. Architecture split, not a restriction.
- The two MIG partition registers read **zero on both 170HX units** while every A100 and the
  Drive carry values. These memory-partition values concern the memory side of MIG and do not
  determine the compute-engine enumeration; the observed one-profile limit is separately
  consistent with an inferred syspipe-disable mask (`0x008203a4 = 0xfe`) and a device-info table
  that lists only GR0. See [MIG: root cause](../frontier/mig.md#root-cause-evidence-for-a-hardware-enumeration-limit-syspipe-fuse-attribution-provisional).

---

## FBPA configuration

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | GA10x | Drive |
|---|---|---|---|---|---|---|---|---|
| `FBPA_NUM_ACTIVE` | `0x009A0164` | `0x0000000a` | `0x0000000a` | `0x0000000a` | `0x0000000a` | `0x0000000a` | `0x00000005`–`0x00000006` | `0x00000008` |
| `FBPA_CFG0_BROADCAST` | `0x009A0200` | `0x07981800` | `0x07981800` | `0x07981800` | `0x07981800` | `0x07981800` | `0x069f9803` / `0x06df9803` | `0x06981800` |
| `FBPA_CFG1_BROADCAST` | `0x009A0204` | `0x02449000` | `0x02449000` | `0x02669000` | `0x02669000` | `0x02779000` | `0x4266b000` etc. | `0x22779000` |
| `FBPA_HBM_CFG0` | `0x009A038C` | `0x000000a7` | `0x000000a7` | `0x000000a7` | `0x000000a7` | `0x000000a7` | `0x000003fe` | `0x000000a6` |

This one table contains the entire memory unlock in miniature:

- **`FBPA_CFG1_BROADCAST` `0x009a0204` reads `0x02449000` on a stock 170HX of either SKU.** The
  A100 40 GB reads `0x02669000` and the A100 80 GB reads `0x02779000`. Those are exactly the two
  values the unlocker writes: `0x02669000` for the 10 GB card (giving 40 GB) and `0x02779000` for
  the 8 GB card (giving 64 GB). **The unlock literally programs the 170HX with an A100's
  addressing depth.** The `[23:16]` tier byte is the field that moves: `0x44` stock,
  `0x66` = 2048 MiB per FBPA, `0x77` = 4096 MiB per FBPA. Note the Drive A100 reads `0x22779000`,
  the 80 GB tier byte with a different high nibble, on a 32 GB card, so the tier byte is
  addressing depth and not capacity.
- **`FBPA_CFG0_BROADCAST` = `0x07981800`** on the 170HX and on all three A100s, `0x06981800` on
  the Drive. A widely circulated annotation claiming `CMP170HX = 0x24490000, A100 = 0x26690000`
  for this register is **wrong**; it is contradicted by live reads on a 170HX (`0x07981800`) and
  on an RTX 3070 (`0x069f9803`). Use the measurement, discard the annotation.
- **`FBPA_HBM_CFG0`**: fields are `dual_rank[0]`, `dual_rank_bank[1]`, `SID_VAL[11]`. The 170HX
  and all three A100s read `0xa7` (bit 0 set, dual rank); the Drive A100 32 GB reads `0xa6`
  (bit 0 clear, single rank). GA10x reads `0x3fe`, a different memory technology with different
  field meanings.

---

## HBM mode registers, ECC control and IEEE 1500

| Register | Address | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | GA10x | Drive |
|---|---|---|---|---|---|---|---|---|
| `FBPA_MRS_0` | `0x009A0300` | `0x00000003` | `0x00000003` | `0x00000003` | `0x00000003` | `0x00000003` | `0x00000025` / `0x00000027` | `0x00000003` |
| `FBPA_MRS_1` | `0x009A0304` | `0x00100000` | `0x00100000` | `0x00100000` | `0x00100000` | `0x00100000` | `0x00100000` | `0x00100000` |
| `FBPA_MRS_2` | `0x009A0334` | `0x002000cf` | `0x002000cf` | `0x002000cf` | `0x002000cf` | `0x00200029` | `0x00200fc0` / `0x002007db` | `0x00200031` |
| `FBPA_MRS_WL_RL` | `0x009A0338` | `0x003000ea` | `0x003000ea` | `0x003000ea` | `0x003000ea` | `0x003000ef` | `0x00300000` / `0x00300010` | `0x003000ef` |
| `FBPA_MRS_8` | `0x009A0320` | `0x00200000` | `0x00200000` | `0x00200000` | `0x00200000` | `0x00200000` | `0x00200000` | `0x00200000` |
| `FBPA_ECC_CTRL` | `0x009A0470` | `0x00000000` | `0x00000000` | `0x00000041` | `0x00000041` | `0x00000041` | `0x20000020` / `0x20000021` | `0x00000041` |
| `FBPA_VEND_ID_C0` | `0x009A0838` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `FBPA_VEND_ID_C1` | `0x009A083C` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `FBPA_TRAINING_STATUS` | `0x009A0974` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` | `0x00000000` |
| `I1500_INSTR` | `0x009A3CB4` | `0x00000f0e` | `0x00000f0e` | `0x0000000f` | `0x0000000f` | `0x0000000f` | `0xbadf5040` | `0x0000000f` |
| `I1500_MODE` | `0x009A3CB8` | `0x00000052` | `0x00000052` | `0x00000008` | `0x00000008` | `0x00000008` | `0xbadf5040` | `0x00000008` |
| `I1500_DATA` | `0x009A3CBC` | `0xde79ffc1` | `0xc631ffc1` | `0x27000000` | `0x1c000000` | `0x3b000000` | `0xbadf5040` | `0x3c000000` |
| `I1500_SHADOW_WIR` | `0x009A3CC0` | `0x0000f0ef` | `0x0000f0ef` | `0x000000f0` | `0x000000f0` | `0x000000f0` | `0xbadf5040` | `0x000000f0` |
| `I1500_SHADOW_WDR` | `0x009A3CC4` | `0xbcf3ff83` | `0x8c63ff83` | `0x4f00f000` | `0x3800f000` | `0x7700f000` | `0xbadf5040` | `0x7800f000` |
| `I1500_STATUS` | `0x009A3CC8` | `0x0000000e` | `0x0000000e` | `0x00000000` | `0x00000000` | `0x00000000` | `0xbadf5040` | `0x00000000` |

Reading these:

- **The 170HX's HBM timing set is identical to the A100 40 GB's.** `FBPA_MRS_2 = 0x002000cf` and
  `FBPA_MRS_WL_RL = 0x003000ea` match both A100 40 GB parts exactly; the A100 80 GB and the Drive
  use a different pair. This is direct evidence that the stacks are being driven with A100-class
  parameters, not a downgraded profile.
- **`FBPA_TRAINING_STATUS = 0x00000000` (`FINISHED`) on every card including both 170HX units.**
  HBM training completes cleanly. Whatever is limiting capacity, it is not a training failure.
- **`FBPA_ECC_CTRL = 0x00000000` on the 170HX** where every A100 and the Drive read `0x00000041`
  (`MASTER_EN[0]` plus `SIDEBAND[6]`). `MASTER_EN` is annotated read-only. This is the concrete,
  register-level face of the ECC fuse: the master enable is not merely off, it is not writable.
- **The IEEE 1500 block is in a different state on the 170HX than on any A100.** `MODE` reads
  `0x52` against `0x08`, `INSTR` reads `0x00000f0e` against `0x0000000f`, `SHADOW_WIR` reads
  `0x0000f0ef` against `0x000000f0`, and `STATUS` reads `0x0e` against `0x00`. The two are
  therefore not in comparable bridge states at probe time, so the 170HX `DATA` and `SHADOW_WDR`
  values should be treated only as a **per-die HBM identifier** (they are two of the 13 registers
  that differ between the two units) and not compared numerically against the A100 column. GA10x
  parts have no readable IEEE 1500 block at these offsets.
- `FBPA_VEND_ID_C0` and `_C1` read zero on all 14 cards, so the HBM vendor identity is not
  available through this path on any of them.

---

## GSP security

| Register | Address | 170HX A | 170HX B | Everything else with data |
|---|---|---|---|---|
| `FUSE_OPT_SECURE_GSP` | `0x0082074C` | `0x00000001` | `0x00000001` | `0x00000001` on all 12 |

The GSP debug-disable fuse is blown on every Ampere card ever probed, including cards with no
other restrictions. It is not a 170HX-specific lock and it does not distinguish this card from an
A100. What it does mean is that no debug path into the GSP exists on any of them, which is why
the unlock had to go through a memory-safety bug in the signature-handling path rather than
through a debug interface. See [Falcon and Booter](../unlock/falcon-and-booter.md).

---

## Per-FBPA `CSTATUS_RAMAMOUNT`

Read at `0x0090020C + n * 0x4000` for n = 0..23. This is the register that reports how much
memory each partition believes it has.

| FBPA | 170HX A | 170HX B | A100 SXM4 40G | A100 PCIe 40G | A100 PCIe 80G | Drive |
|---|---|---|---|---|---|---|
| 00 | `0x00000200` | `0xbadf2010` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 01 | `0x00000200` | `0xbadf2010` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 02 | `0x00000200` | `0x00000200` | `0xbadf2011` | `0x000007ff` | `0x00000fff` | `0xbadf2011` |
| 03 | `0x00000200` | `0x00000200` | `0xbadf2011` | `0x000007ff` | `0x00000fff` | `0xbadf2011` |
| 04 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 05 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 06 | `0x00000200` | `0xbadf2013` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 07 | `0x00000200` | `0xbadf2013` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 08 | `0x00000200` | `0x00000200` | `0xbadf2014` | `0x000007ff` | `0x00000fff` | `0xbadf2014` |
| 09 | `0x00000200` | `0x00000200` | `0xbadf2014` | `0x000007ff` | `0x00000fff` | `0xbadf2014` |
| 10 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 11 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 12 | `0x00000200` | `0x00000200` | `0x000007ff` | `0xbadf2016` | `0xbadf2016` | `0xbadf2016` |
| 13 | `0x00000200` | `0x00000200` | `0x000007ff` | `0xbadf2016` | `0xbadf2016` | `0xbadf2016` |
| 14 | `0xbadf2017` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 15 | `0xbadf2017` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 16 | `0xbadf2018` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 17 | `0xbadf2018` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 18 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 19 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 20 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 21 | `0x00000200` | `0x00000200` | `0x000007ff` | `0x000007ff` | `0x00000fff` | `0x000007ff` |
| 22 | `0x00000200` | `0x00000200` | `0x000007ff` | `0xbadf201b` | `0xbadf201b` | `0xbadf201b` |
| 23 | `0x00000200` | `0x00000200` | `0x000007ff` | `0xbadf201b` | `0xbadf201b` | `0xbadf201b` |

The floorswept partitions line up exactly with the `FUSE_FBPA_DISABLE` masks on every card, and
the sentinel's low byte encodes the FBP that owns the pair (`0xbadf2010` for FBPAs 0 and 1,
`0xbadf2013` for 6 and 7, `0xbadf2017` for 14 and 15, `0xbadf2018` for 16 and 17). That is a
second, independent confirmation of the floorsweep decode.

Consumer and professional GA10x parts, all with 6 FBPAs: A10 `0x00000efe`, A5000 `0x00000ffe`,
A6000 `0x00001ffe`, RTX 3080 `0x00000800` with `fbpa03` swept, RTX 3080 Ti `0x00000800`,
RTX 3090 `0x00001000`, RTX 3090 Ti `0x00000ffe`, and `0xbadf1100` (not decoded) for FBPAs 6
through 23.

> [!NOTE]
> **Open problem: the exact `CSTATUS_RAMAMOUNT` encoding**
>
> The 170HX reads exactly `0x200` = 512 and 20 × 512 MiB = 10240 MiB, which is right. But the
> A100 40 GB reads `0x7ff` = 2047 and the survey treats it as 2048 MiB per partition, while the
> RTX 3080 reads `0x800` = 2048 for the same nominal 2048 MiB. Observationally, every part with
> `FUSE_ECC_EN = 1` reports slightly **less** than a power of two (`0x7ff`, `0xfff`, `0xefe`,
> `0xffe`, `0x1ffe`) and the ECC-disabled parts report an exact power of two (`0x200`, `0x800`,
> `0x1000`), with the RTX 3090 Ti (`0xffe`, ECC fuse `0`) as the one exception. Whether the
> field is post-ECC usable capacity, an n-minus-one encoding, or something else is not settled
> anywhere in the corpus. It does not affect the unlock, which writes `CFG1` and the local
> memory range rather than this register.

## Per-FBPA `CFG0`

Read at `0x00900200 + n * 0x4000`. Every live partition on both 170HX units reads
**`0x07981800`**, identical to the broadcast register and identical to every live partition on all
three A100s. Disabled partitions return the same `0xbadf20xx` sentinels as above, at the same
indices. The Drive A100 reads `0x06981800` on its live partitions. GA10x parts read `0x069f9803`
or `0x06df9803`.

The practical point: **there is no per-partition variation on this card**. Any theory that
individual FBPAs are configured differently, or that one partition holds a strap or a
per-stack override, is refuted by 20 identical reads per unit across two units.

---

## What is overridable, and what is not

| Restriction | Fuse | 170HX value | Override path | Status |
|---|---|---|---|---|
| Arithmetic throughput | `FUSE_SS_*` `0x00820224`, `0x0082059C`, `0x008207D4`–`0x008207EC` | `0x5` × 8, DP `0x1` | Feature override `0x0082381C` / `0x00823820` after opening PLM `0x00823804` | **Solved and shipping.** Survives FLR |
| Memory capacity | none directly; `FBPA_CFG1` `0x009a0204` plus the MMU local memory range | `0x02449000` | Host write after opening FBPA PLM `0x009a0148` | **Solved and shipping.** Does **not** survive FLR |
| PCIe Gen 2 | `FUSE_PCIE_GEN23_DIS` `0x0082057C` | `0x00000001` | Fuse shadow write **fails**; Gen2 reached via `CYA_0`/`LINK_CONFIG_0`/XP3G/`PRIV_MISC_1` | Shipped in `master` since 2026-07-29 |
| PCIe Gen 3 | `FUSE_PCIE_GEN3_DIS` `0x00820580` | `0x00000001` | none found | **Open.** Assessed as needing a GSP patch |
| PCIe Gen 4 | `FUSE_PCIE_MAGIC_D` `0x00820520` bit 25 | `0x16680000` | Writability never tested | **Open** |
| PCIe width | none. `FUSE_PCIE_LANE_DIS` = `0` | `0x00000000` | Solder 24 × 0402 capacitors | Physical mod only |
| NVLink | `FUSE_NVLINK_DIS` `0x00820684` | `0x00000007` | No FEAT_OVR register exists; `CTRL_OPT_NVLINK` inert | **Closed.** No known path |
| ECC | `FUSE_ECC_EN` `0x00820228` | `0x00000000` | none found; `FBPA_ECC_CTRL.MASTER_EN` read-only | **Closed.** No known path |
| PCI device ID | `FUSE_DEVID_SW_OVR_DIS` `0x00820584` | `0x00000001` | none | **Closed** by fuse |
| Floorswept GPCs / FBPs | `FUSE_GPC_DISABLE`, `FUSE_FBP_DISABLE`, `FUSE_FBPA_DISABLE` | per die | `CTRL_OPT_*`, inert because `FUSE_EN_SW_OVERRIDE` = `0` | **Untried.** Never demonstrated |
| Everything, permanently | `FUSE_FEAT_OVR_DIS` `0x008203F0` | `0x00000000` | not blown | The reason any of this works |

---

## How the survey's conclusion was overturned

The May 2026 fuse table ends with a bolded verdict:

> **Host-level register write approach CONFIRMED DEAD for compute unlock**

That verdict was correct for what it tested, and its diagnosis of *why* remains completely valid
today:

- `FEAT_OVR_PLM` and `FEAT_OVR_ECC_PLM` both read `0xffffff8f`, meaning level-3 access only, and
  a CPU driver runs at level 0.
- `FUSE_QUADRO_WR_SEC` = `1` seals that PLM.
- `FUSE_EN_SW_OVERRIDE` = `0` disables the `CTRL_OPT` override table.
- `FECS_FEAT_OVERRIDE` reads return `0xbadf5040`.

What the document lacked was a way to reach level 3, and it said so in the same breath: its own
key findings note that `FUSE_FEAT_OVR_DIS` = `0` means the "override path [is] architecturally
available (needs Falcon HS)". Six to eight weeks later, that is exactly what happened. The
shipping unlock:

1. enlarges the driver-side GSP signature buffer (`pSignatureMemdesc`) from
   `NV_ALIGN_UP(pGspFw->signatureSize, 256)` to `0x0000f800` bytes, which nothing rejects;
2. plants a crafted Falcon payload in it and re-fires Booter Load once per target to open four
   privilege level masks: `0x001fa7cc` to `0xfffff0ff`, `0x009a0148`, `0x001fa7c4` and
   `0x00823804` to `0xffffffff`;
3. then performs **plain host BAR0 writes**, `GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U)` and
   `GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U)`.

So host writes were never the problem. The privilege level was. The correct restatement of the
survey's conclusion is: *host writes to the feature-override block are dead **while the PLM is
closed**, and nothing in the fuse array prevents them once it is open*. See
[How it works](../unlock/how-it-works.md) and
[Privilege level masks](../unlock/privilege-level-masks.md).

The one thing the fuse survey got exactly right and that has never been dislodged is which single
zero the whole enterprise depends on.

---

## Reproducing the survey

The probe is read-only. It mmaps BAR0 through sysfs and issues 32-bit loads. It does not write
anything and it does not need the NVIDIA driver.

```bash
# The published probe, run against a 10 GB card
sudo ./probe.sh 10de:2082

# Output lands in /tmp/mmio-probe-<epoch>/ :
#   probe.log, lspci.txt, nvidia-smi.txt, gpu-summary.csv, registers.json
```

To read a single register without the script:

```bash
sudo python3 - <<'EOF'
import mmap, struct
BDF  = '0000:81:00.0'          # your card
ADDR = 0x00820684              # FUSE_NVLINK_DIS
with open(f'/sys/bus/pci/devices/{BDF}/resource0', 'rb') as f:
    bar = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
    print(f'0x{ADDR:08x} = 0x{struct.unpack_from("<I", bar, ADDR)[0]:08x}')
EOF
```

Requirements and caveats:

- Root, and inside a container it must be privileged. The script's header comment promises a
  `/dev/mem` fallback, but the published code does not implement one: if `resource0` is missing
  it logs an error and exits with status 2.
- Context matters. Several registers (`SM_ISSUE_RATE_MODIFIER`, the SKED block) return
  `0xbadf1201` when no driver is loaded and read cleanly once it is. `probe.sh` calls
  `nvidia-smi` before it reads, so it brings the driver up itself. Record which state you were in
  or the numbers are not comparable.
- Cards with a small BAR0 (the A16, at 256 KB) simply cannot reach anything at or above
  offset `0x40000`.
- If you add a card to the cohort, report `lspci -nn` for the same bus address in the same
  capture. Several unresolved items on this page exist only because a probe output and a device
  ID were never published together.

---

## Gaps

Things this page cannot tell you, listed so that nobody has to rediscover the hole.

1. **No 8 GB (`0x20C2`) card was ever put through this survey.** Every 170HX column here is a
   10 GB card. The `NV_PTOP_FS4` question and the predicted `DEVIDB = 0x2102` both hang on this.
2. **`FUSE_FB_FALCON_PRI_DIS` (`0x00820670`)** is read by the probe and dropped from the
   published table. Its value on the 170HX is unknown, and it decides whether a Falcon can touch
   FB PRI registers at all.
3. **`PTOP_SCAL_FBPA_PER_FBP` (`0x00022458`)** is likewise read and not tabulated. Expected `2`
   on GA100 from the FBP-to-FBPA arithmetic above, but unrecorded.
4. **No TPC-level floorsweep register is probed**, so the five swept TPCs implied by 70 SMs
   against 5 enabled GPCs are invisible here.
5. **`FEAT_READOUT_1`'s field layout is undocumented.** The value correlates perfectly with fuse
   state across all 14 cards, but nobody has decoded `0x016db6ed` into nine per-unit fields.
6. **`FEAT_OVR_ECC_PLM` reads `0x0000abcf` on exactly one card** (A100 SXM4 40 GB) against
   `0xffffff8f` on the other 13. Unexplained, never re-read.
7. **No write probe has ever been run on `CTRL_OPT_*`.** The strong prior says the writes will be
   dropped. The corpus cannot currently state that they were tried and failed, only that nobody
   tried.
8. **The engineering-sample column was never filled in.** The card suffered a thermal event on
   2026-05-05 and its status is unknown.

## See also

- [GA100 silicon](ga100-silicon.md) for the die and its topology
- [Memory subsystem](memory-subsystem.md) for what `CFG1` and the FBPA aperture actually do
- [PCIe subsystem](pcie-subsystem.md) for the speed and width restrictions in detail
- [NVLink hardware](nvlink-hardware.md) for the NVLink fuse and the physical connectors
- [Privilege level masks](../unlock/privilege-level-masks.md) for the PLM model and the four-entry table
- [Compute throttle](../unlock/compute-throttle.md) for the SS0/SS1 write
- [Register reference](../unlock/register-reference.md) and
  [Register index](../appendix/register-index.md) for lookup by address
- [Methodology](../appendix/methodology.md) for how the values on this wiki were sourced and rated
