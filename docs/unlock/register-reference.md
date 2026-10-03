# Register reference

## What this page covers

Every hardware register that anyone in this project has read, written, or argued about on the
NVIDIA CMP 170HX (GA100), organised by functional block, with its address, its name, what it
does, its stock value, its unlocked value where one exists, whether a
[privilege level mask](privilege-level-masks.md) gates host writes to it, whether its contents
survive a Function Level Reset (FLR), and which patch or tool touches it.

The single most important fact on this page: **the entire shipping unlock is four privilege
level mask opens followed by four ordinary host register writes.** Everything else here is
context, diagnostics, or unreleased work.

| Step | Address | Value written | Mechanism |
|---|---|---|---|
| PLM open 0 | `0x001fa7cc` | `0xfffff0ff` | Booter Load re-fire |
| PLM open 1 | `0x009a0148` | `0xffffffff` | Booter Load re-fire |
| PLM open 2 | `0x001fa7c4` | `0xffffffff` | Booter Load re-fire |
| PLM open 3 | `0x00823804` | `0xffffffff` | Booter Load re-fire |
| Unlock write 0 | `0x0082381c` | `0x88888888` | plain host `GPU_REG_WR32` |
| Unlock write 1 | `0x00823820` | `0x00000008` | plain host `GPU_REG_WR32` |
| Unlock write 2 | `0x009a0204` | `0x02779000` (8 GB card) / `0x02669000` (10 GB card) | plain host `GPU_REG_WR32` |
| Unlock write 3 | `0x00100ce0` | `0x0000020B` (8 GB card) / `0x0000028A` (10 GB card) | plain host `GPU_REG_WR32` |

Everything above is read directly out of
`driver/patches/0001-sec2-postbl-plm-ss-cfg.patch` on shipping `master`. See
[How the unlock works](how-it-works.md) for the narrative version and
[Driver patches](driver-patches.md) for the patch-by-patch breakdown.

---

## Reading conventions

### Addresses versus values

This is the single most common source of confusion in the archive, so it gets its own rule:
**everything in a `Address` column is a BAR0 byte offset; everything in a `Value` column is
32 bits of data.** Several unlock values are themselves plausible-looking addresses, and
several payload slots contain Falcon IMEM addresses that are *not* BAR0 registers at all.
The [Numbers that are not BAR0 addresses](#numbers-that-are-not-bar0-addresses) section at the
bottom lists every trap of this kind.

All BAR0 offsets on this page are absolute from the start of BAR0 (region 0). To read one by
hand:

```bash
# BAR0 offset 0x009a0204 on the GPU at 0000:05:00.0
sudo dd if=/sys/bus/pci/devices/0000:05:00.0/resource0 \
        bs=4 count=1 skip=$((0x009a0204 / 4)) 2>/dev/null | xxd -e -g4
```

### Column meanings

- **PLM-gated**: whether a PL0 (ordinary host) write to this register is silently dropped until
  a privilege level mask is opened from Heavy-Secure (HS) Falcon context. "Silently" is literal:
  an early pipeline logged `Write failed - wrote 0x2779000, read 0x2449000` three times with no
  error signalled anywhere.
- **FLR**: whether the value survives `echo 1 > /sys/bus/pci/devices/<bdf>/reset`. Only three
  registers in the whole map are known to survive, and that asymmetry is the reason the compute
  unlock shipped weeks before the memory unlock.
- **Touched by**: the shipping patch, an unreleased branch patch, or a clean-room tool. "read
  only" means something in the project reads it for diagnostics but nothing writes it.

### Sentinel values

A read that returns one of these is not data:

| Sentinel | Meaning |
|---|---|
| `0xbadf0000` (masked with `0xffff0000`) | generic PRI poison, tested for by the Booter's own `csb_read` |
| `0xbadf1100` | PRI poison seen at `0x85080` / `0x85084` from the SEC2 injection point, and at `0x00021c14` / `0x0000a800` (registers absent on GA100) |
| `0xbadf1201` | returned by `0x00504204` and the whole `0x00407xxx` SKED block on a 170HX with no driver loaded |
| `0xbadf2010`, `0xbadf2011`, `0xbadf2013`, `0xbadf2017`, `0xbadf2018` | CSTATUS read of a floorswept (disabled) FBPA |
| `0xbadf5040` | privilege-violation sentinel; `0x00409664` / `0x00409668` return it on every Ampere card ever probed, and the XVE shadow returns it to unprivileged readers |
| `0xbadf5108` | host PL0 read of a secure PLM |
| `0xdead5ec1` | PLM-poisoned read of `0x001180f8`; note bit 26 already reads as 1 in this pattern, producing a false BOOT_STAGE_3 "DONE" |
| `0xffffffff` on a whole aperture | BAR0 mapping is dead (card fell off the bus), not an unlocked PLM |

---

## SEC2 Falcon (BAR0 + `0x840000`)

SEC2 is the security coprocessor whose signed `booter_load` ucode the exploit hijacks. It is a
Falcon v4 core, not RISC-V (`FALCON_HWCFG2` bit 10 reads 0). See
[Falcon and Booter](falcon-and-booter.md).

| Address | Name | Function | Notes |
|---|---|---|---|
| `0x00840040` | `FALCON_MAILBOX0` | status word after a Booter run | `0x31` on every payload-carrying run; readback of the target register is the only valid verdict |
| `0x00840044` | `FALCON_MAILBOX1` | second mailbox | |
| `0x0084007c` | `SFTRESET` | soft reset; write 1 and read back | only if `SCTL` HSMODE (bit 1) is set |
| `0x00840084` | `FALCON_RM` | Falcon resource-manager scratch | |
| `0x008400f4` | `FALCON_HWCFG2` | bit 10 = RISCV | reads 0 on SEC2 (Falcon core) |
| `0x00840100` | `FALCON_CPUCTL` | bit 1 = STARTCPU pulse, bit 4 = HALTED (RO) | |
| `0x00840104` | `FALCON_BOOTVEC` | boot vector | |
| `0x0084010c` | `FALCON_DMACTL` | poll until scrub bits `0x6` clear | `0xffffffff` reads mean the window is not responding yet, not failure |
| `0x00840110` | `FALCON_DMATRFBASE` | DMA base | |
| `0x00840114` | `FALCON_DMATRFMOFFS` | DMEM/IMEM offset | |
| `0x00840118` | `FALCON_DMATRFCMD` | DMA command | |
| `0x0084011c` | `FALCON_DMATRFFBOFFS` | FB offset | |
| `0x00840128` | `FALCON_DMATRFBASE1` | DMA base high | |
| `0x00840180` / `0x00840184` | `IMEMC` / `IMEMD` | IMEM port, auto-increment, per-256-byte tags | secure tag bit is `1 << 28`; used to read the loaded booter back out over range 0..`0x8700` |
| `0x00840240` | `SCTL` | secure control; HSMODE = bit 1, `AUTH_EN` = `1 << 14` | observed `0x3000` to `0x3002` after an engine reset |
| `0x00840284` | SEC2 `DMEM_PLM` | DMEM privilege mask | `0xff` (fully open) in LS mode |
| `0x008403c0` | `FALCON_ENGINE` | bit 0 = RESET; pulse 1 then 0 | engine-reset gate is `(resetPLM & 0x77) == 0x77` |
| `0x008403c4` | **SEC2 reset PLM** | decides whether SEC2 can be reset again | `0xff` clean, `0xdf` after a successful `booter_unload`, `0x8f` after `secure_teardown` (blocks SFTRESET). `reset_allowed = {0xff, 0xdf}`. **Cleared to `0xff` by FLR.** Read by every clean-room fire tool as a readiness gate |
| `0x00840480` / `0x00840484` | SEC2 post-fire state | `0` to `0x1` and `0` to `0x11100` | HS-exit side effect, never restored |
| `0x00840530` | `SCP_P2PRX` | poll bit 3 during driverless reset | |
| `0x008411ec` | `KFUSE_CTL` | poll bit 0 set, bit 1 clear | |

> [!NOTE]
> **Falcon-internal addresses are a different space**
>
> Inside the running Booter, `I[0x1c100]` / `I[0x1c200]` / `I[0x1c000]` are the BAR0-master
> address, data and command ports in Falcon CSB space (`0x800000f2` = write,
> `0x800000f1` = read), and `I[0x1c300]` is the watchdog, seeded with `0x1312d00`
> (20,000,000). `I[0x12000]`, `I[0x12100]`, `I[0x12400]` and `I[0x12600]` are Falcon-local
> aperture/PLM words the Booter programs early in `main()`. None of these are BAR0 offsets and
> none can be poked from the host. Likewise `0x9100` is `FALCON_CSBERRSTAT`, whose bit 31 is
> the fail-closed fault flag after every CSB access. See [The ROP chain](rop-chain.md).

## GSP RISC-V and BSI secure scratch (BAR0 + `0x110000` / `0x118000`)

| Address | Name | Function | Notes |
|---|---|---|---|
| `0x00110624` | GSP `FBIF_CTL` | aperture control | Booter `reg_init (0x68ed)` writes `0x90` (`ALLOW_PHYS_NO_CTX` bit 7 plus bit 4) |
| `0x00110684` | GSP FBIF companion | | written `1` by `reg_init` |
| `0x0011126c` | GSP RISC-V companion | | written `1` by `reg_init` |
| `0x00111240` | `RISCV_STATUS` | GSP core status | `0x35` = RISC-V active; `0x0` = never started |
| `0x00111268` | `RISCV_CPUCTL` | GSP core control | |
| `0x00118244` / `0x00118248` | WPR staging pair | read then zeroed by `booter_load_wpr_main (0x22ba)` | |
| `0x0011824c` / `0x00118250` | memcfg handoff | written by `memcfg_program (0x79cc)`; `memcfg_apply_poll` runs only if `0x11824c` bit 0 is set | |
| `0x001180f0` | AON LMR shadow | always-on shadow of the memory-range value | **reverts on FLR**; not a persistence lever |
| `0x001180f8` | `NV_PGC6_BSI_SECURE_SCRATCH_14` | bit 26 = `BOOT_STAGE_3_HANDOFF` (INIT 0, DONE 1) | set GPU-side by SEC2 in HS context; the host driver only polls it. Boot hang `0x65` is that poll timing out. The Booter's own entry checks require bits [31:28] == 0 (else error `0x29`) and bits [27:24] == 0 (else `0x88`). **The shipping chain never writes this register.** |
| `0x001182d0` | AON secure scratch | | reachable at PL3 |
| `0x00118f78` | auxiliary scratch | | reads `0x00000000` on every card surveyed |

> [!NOTE]
> **Open problem: the correct `0x001180f8` handoff value**
>
> `0x11000000`, `0x13100000` and `0x17100000` were all proposed. `0x17100000` was measured to
> satisfy both the Booter's `0x29` check and the host DONE poll, but writing it also caused
> both `FBPA_008` and `FBPA_00C` to fail their Booter PLM opens. The shipping chain sidesteps
> the whole question by never touching the register. See
> [Open questions](../frontier/open-questions.md).

## WPR block (`0x001fa7xx` / `0x001fa8xx`)

Write-Protected Regions. The unlock must save and restore WPR2 around every Booter re-fire,
because a second `booter_load` otherwise aborts with "WPR2 already up".

| Address | Name | Function | Stock | Touched by |
|---|---|---|---|---|
| `0x001fa7c4` | `WPR_PLM` | privilege mask over the WPR region registers | `0x0004cb8f` | shipping patch 0001, PLM index 2, opened to `0xffffffff` |
| `0x001fa7c8` | `MMU_LOCK` PLM | write nibble `0x8` = L3/HS only | `0x0004cb8f` | read only |
| `0x001fa7cc` | `WPR_CFG_PLM` (WPR mask PLM) | privilege mask over the WPR allow-masks | `0x0004cb8f` | shipping patch 0001, PLM index 0, opened to **`0xfffff0ff`**, not `0xffffffff` |
| `0x001fa814` | WPR read-allow mask | mode field in bits [7:4] | | Booter `fbif_set_bit800 (0x8307)` sets bit `0x800` under mask `0x0ffff8ff` |
| `0x001fa818` | WPR write-allow mask | as above | | same |
| `0x001fa81c` | `WPR1_ADDR_LO` | value in bits [31:4], `<< 12` | | cleared by the clean-room re-fire chain |
| `0x001fa820` | `WPR1_ADDR_HI` | | | same |
| `0x001fa824` | `WPR2_ADDR_LO` | value in bits [31:4], `<< 12` | empty/INIT = `0x0fffffff` | **saved and re-armed by the shipping patch before every Booter attempt**; clean-room teardown value is `0x1ffffe00` |
| `0x001fa828` | `WPR2_ADDR_HI` | `HI = 0` makes `kgspIsWpr2Up()` return false | empty/INIT = `0` | same |
| `0x001fa82c` / `0x001fa830` | memlock range LO / HI | | `0x1ffffff0` / `0x00000000` post-AHESASC (empty) | read only |

Measured WPR2 values in the wild: `0x02777000` carved on a 10 GB card, in both legs of the
10 GB versus 40 GB A/B;
`[0x1ffffe00, 0x027fee00]` after a 40 GB clean-room fire; `07f68000/07fefe00` on a PG199
reference board. RM re-allocates WPR2 at a fresh FB address every boot, which is why a
statically baked WprMeta finalizes at `0xf0000000` (failure) while live values give
`0x11000000`.

---

## Privilege level masks: the complete catalog

A PLM is a 32-bit word of nibble-encoded permissions guarding another register or register
group. `0xffffff8f` is the common locked state (level 3 only for writes);
`0xffffffff` is fully open; `0x0004cb8f` is the locked state of the WPR pair. See
[Privilege level masks](privilege-level-masks.md) for the encoding.

### Written by shipping `master` (four entries, in this order, up to two attempts each)

| Index | Address | Label in code | Value written | Notes |
|---|---|---|---|---|
| 0 | `0x001fa7cc` | `WPR_CFG` | `0xfffff0ff` | the exception: **not** `0xffffffff` |
| 1 | `0x009a0148` | `FBPA` | `0xffffffff` | also the built-in fallback payload target |
| 2 | `0x001fa7c4` | `WPR` | `0xffffffff` | |
| 3 | `0x00823804` | `FEAT` | `0xffffffff` | the only AON entry; survives FLR |

Each attempt re-arms `0x001fa824` / `0x001fa828` from values saved before the loop, refills the
whole `0xf800`-byte payload for that one (address, value) pair, fires Booter Load, and reads the
target back. Success is defined purely by readback equality. Timing from a 2026-07-19 dmesg
capture on an 8 GB card: PLM[0] at 11.32 s, PLM[1] 11.50 s, PLM[2] 11.68 s, PLM[3] 11.86 s,
about 180 ms per Booter pass.

### Added by the Gen2-family branches (nine entries total)

Branches `Gen2`, `debug-gen2`, `far` and `deced` carry an identical modified patch 0001 whose
`plmTable[]` has nine rows and whose loop bound is `plmIdx < 9`. The five extra entries, all
written `0xffffffff`:

| Address | Label | Purpose |
|---|---|---|
| `0x00088ff4` | `XVE` | XVE config-space shadow PLM; needed because host reads of the PCIe shadow otherwise return `0xbadf5040` |
| `0x00088ab4` | `XVE_B` | second XVE capability PLM |
| `0x00088ff8` | `XVE_C` | third XVE capability PLM |
| `0x00823b00` | `FEAT2` (row-remapper PLM) | stock `0xffffff8f`; one in-HS sweep read it `0xffffffff` after FLR, so it may be AON, but opening it did not make geometry persist |
| `0x008200fc` | `OPT_PLM` (aliases: `FUSE_SS_PLM`) | already reads `0xffffffff` on stock cards in one sweep and `0x000003ff` in another; **never written by shipping master** |

> [!WARNING]
> **Experimental**
>
> The nine-entry table exists only on unreleased branches. All nine were reported succeeding
> on the first attempt on both GPUs of one two-card rig. See [PCIe Gen2](pcie-gen2.md).

### The FB-geometry PLM set (clean-room tools only)

`refire_chain_v9.py` uses `FB_GEO_PLMS = [0x00100b10, 0x009a0148, 0x009a014c, 0x009a0008,
0x009a000c]`. The earliest HS recipes used a six-entry variant adding `0x00100b38`. Opening any
of them requires L3.

**None of these move FB geometry into the always-on domain.** A dedicated survival script
(`geo_flr_survival_map_20260716.sh`) opened all six plus `FUSE_SS_PLM` and found CFG1, CSTATUS
and LMR still reverting on FLR. That negative result is why the shipping design re-applies
geometry inside the GSP boot path on every module load instead of trying to make it stick.

### The 26-register PLM survey

`nuke.sh` classified this candidate set with a cold-boot baseline plus nine FLR cycles:

```text
0x008200D0  0x008200D4  0x008200D8  0x008200DC  0x008200E0  0x008200E4
0x008200E8  0x008200EC  0x008200F0  0x008200F4  0x008200FC
0x00823800  0x00823804  0x00823B00
0x009A0008  0x009A000C  0x009A0148  0x009A014C  0x009A0168  0x009A03F0
0x009A0554  0x009A0BFC
0x00100B10  0x00100B38  0x00100B84  0x00100B9C
```

Measured readbacks from a 580.159.04 sweep after two FLRs with no driver loaded:

| Registers | Value | Reading |
|---|---|---|
| `0x8200D4`, `D8`, `E0`, `E4`, `E8`, `EC`, `F0`, `F4`, `FC`, `0x823800`, `0x823804`, `0x823B00` | open | reported UNLOCKED |
| `0x8200D0`, `0x8200DC` | `0xffffff8f` | locked |
| `0x9A0008`, `0x9A000C`, `0x9A0148`, `0x9A014C`, `0x9A03F0` | `0xffffff8f` | locked |
| `0x9A0168`, `0x9A0554`, `0x00100B9C` | `0xffffffcf` | locked |
| `0x9A0BFC` | `0x00000000` | |
| `0x00100B10`, `0x00100B38` | `0xffffff8f` | locked |
| `0x00100B84` | `0xffffff88` | locked |

After a **stock** driver load following an unlock, only `0x00823804` stays open. `0x001fa7cc`
and `0x001fa7c4` re-lock to `0x0004cb8f`, `0x009a0148` reads `0xffffff8f`, and host write tests
against `0x009a0008`, `0x00100b10` and `0x00100b38` read `0xfffffe8e`.

---

## Memory geometry: FBPA and MMU

Full narrative in [Memory geometry](memory-geometry.md); hardware background in
[Memory subsystem](../hardware/memory-subsystem.md).

### The two registers the unlock actually writes

| Address | Name | Function | Stock (both SKUs) | 8 GB card unlocked | 10 GB card unlocked | PLM-gated | FLR |
|---|---|---|---|---|---|---|---|
| `0x009a0204` | `NV_PFB_FBPA_CFG1` (broadcast) | addressing depth per memory partition | `0x02449000` | `0x02779000` | `0x02669000` | yes, via `0x009a0148` | **no** |
| `0x00100ce0` | MMU local memory range (LMR) | total FB size seen by the MMU | `0x00000208` (8 GB) / `0x00000288` (10 GB) | `0x0000020B` | `0x0000028A` | yes | **no** |

Both are written unconditionally by patch 0001 immediately after SS0/SS1, selected at runtime
from `pGpu->idInfo.PCIDeviceID >> 16`:

```c
if (devId == SEC2_POSTBL_TIMING_CMP_170HX_8GB_PCI_DEVICE_ID) {   /* 0x20C2 */
    cfg1Value = 0x02779000U;  // 8GB card: 64GB unlock
    lmrValue  = 0x0000020BU;
} else {
    cfg1Value = 0x02669000U;  // 10GB card: 40GB unlock
    lmrValue  = 0x0000028AU;
}
GPU_REG_WR32(pGpu, 0x0082381cU, 0x88888888U);
GPU_REG_WR32(pGpu, 0x00823820U, 0x00000008U);
GPU_REG_WR32(pGpu, 0x009a0204U, cfg1Value);
GPU_REG_WR32(pGpu, 0x00100ce0U, lmrValue);
```

**CFG1 encoding.** The tier byte at bits [23:16] is the lever: `0x44` stock (12 row bits,
512 MiB per FBPA), `0x66` (14 row bits, 2048 MiB per FBPA), `0x77` (15 row bits, 4096 MiB per
FBPA). Total capacity is addressing depth times the fuse-determined active-FBPA count; CFG1 does
not change how many FBPAs exist. The probe-catalog field decode is `SUBP[1:0]`, `COL[15:12]`,
`ROWA[19:16]`, `BANK[25:24]`; on every HBM part observed, COL stays `0x9` and BANK stays `0b10`.
GDDR6 parts read different COL nibbles, which is why the `0x9` is a memory-type constant and not
a "5 stacks" flag.

`0x02779000` is literally the stock A100 PCIe 80 GB CFG1 word, and `0x02669000` is the stock
A100 PCIe 40 GB and SXM4 40 GB word. The unlock restores genuine A100 geometry rather than
inventing constants.

**LMR encoding.** `size_MiB = MAG[9:4] << SCALE[3:0]`, equivalently
`bytes = MAG << (SCALE + 20)`. MAG is constant per SKU (32 on the 8 GB card, 40 on the 10 GB
card) and equals twice the active-FBPA count. SCALE is what the unlock changes.

| LMR | MAG | SCALE | Decodes to | Status |
|---|---|---|---|---|
| `0x00000208` | 32 | 8 | 8192 MiB | stock 8 GB card |
| `0x00000288` | 40 | 8 | 10240 MiB | stock 10 GB card |
| `0x0000020B` | 32 | 11 | 65536 MiB | shipping 8 GB profile |
| `0x0000028A` | 40 | 10 | 40960 MiB | shipping 10 GB profile |
| `0x0000028B` | 40 | 11 | 81920 MiB | inert metadata on the `80` branch, but fired for real by the clean-room refire script, see below |
| `0x0000020A` | 32 | 10 | 32768 MiB | stock on the PG199 Drive reference board |

**CFG1 alone is not enough.** A controlled three-way comparison: no memory writes gives CPU-RM
failure `0x24` (`kbusVerifyBar2`); the 40 GB CFG1 with a stock 10 GB LMR still gives `0x24`;
CFG1 plus a matched LMR reaches `0x25` (StateLoad). GSP-RM additionally treats the LMR as
master: with CFG1 at 40 GB but LMR left at `0x288`, `kgspBootstrap` reverted CSTATUS from
`0x800` back to `0x200`.

### FBPA broadcast aperture (`0x009a0000` to `0x009a3fff`)

| Address | Name | Stock / observed | Notes |
|---|---|---|---|
| `0x009a0008` | FB-geometry PLM | `0xffffff8f` | in the clean-room `FB_GEO_PLMS` list |
| `0x009a000c` | FB-geometry PLM | `0xffffff8f` | same |
| `0x009a0148` | **FBPA PLM** | `0xffffff8f` | opened to `0xffffffff` by shipping patch 0001 |
| `0x009a014c` | FB-geometry PLM | `0xffffff8f` | clean-room list |
| `0x009a0164` | `FBPA_NUM_ACTIVE` (`NUM_ACTIVE_FBPS`) | `0x00000008` on the 8 GB card | |
| `0x009a0168` | PLM candidate | `0xffffffcf` | survey only |
| `0x009a0200` | `FBPA_CFG0_BROADCAST` | `0x07981800` on 170HX and A100 40G/80G; `0x06981800` on a reference GA100/Drive part | identical on every live per-FBPA instance |
| `0x009a0204` | `FBPA_CFG1_BROADCAST` | see above | reference GA100 and A100 32 GB Drive read `0x22779000`, differing from the CMP target only in bit 29 |
| `0x009a020c` | `FBPA_CSTATUS` broadcast | `0x00001000` on unlocked 170HX vs `0x00000fff` on A100 80 GB | |
| `0x009a0224` | `TIMING1` | `0x12050d12` (R2W 18, W2R 13, R2P 5, W2P 18) | programmed CONFIG value |
| `0x009a0290` | `CONFIG0` | `0x1255b93c` | bit 31 `USE_TIMING_REGS` = 0 |
| `0x009a0294` | `CONFIG1` | `0x38d4841b` | CL 27, WL 8, RD_RCD 18, WR_RCD 13, QPOP_OFFSET 14 |
| `0x009a0298` | `CONFIG2` | `0x88130b11` | tWR 19, W2R_BUS 8, R2W_BUS 8, RPRE 1, WPRE 1, CDLR 11 |
| `0x009a02b0` | `TIMING0_GEN` | tRC 60, tRFC 441, tRAS 42 | generated shadow, the one actually in force |
| `0x009a02b4` | `TIMING1_GEN` | R2W 29, W2R 20, W2P 28 | |
| `0x009a02b8` | `TIMING2_GEN` | RD_RCD 18, WR_RCD 13, RRD 6 | |
| `0x009a02c0` | `TIMING4_GEN` | FAW 21 | raw `TIMING4` holds a stale FAW of 40 |
| `0x009a02d8` | `TIMING9_GEN` | CCDL 4, CCDS 2 | |
| `0x009a02e0` | `TIMING16_GEN` | RP 18 | |
| `0x009a0300` | `FBPA_MRS_0` | `0x00000003` on 170HX / A100 / Drive A100 | GDDR6 parts differ |
| `0x009a0304` | `FBPA_MRS_1` | `0x00100000` on every card | |
| `0x009a0320` | `FBPA_MRS_8` (MR8 Density) | `0x00200000` on all 15 cards | **not** the capacity restriction |
| `0x009a0334` | `FBPA_MRS_2` | `0x00200019` (8 GB CMP), `0x002000cf` (10 GB CMP and A100 40G) | |
| `0x009a0338` | `FBPA_MRS_WL_RL` | `0x003000eb` (8 GB CMP), `0x003000ea` (10 GB CMP) | |
| `0x009a038c` | `FBPA_HBM_CFG0` | `0x000000a7` on 170HX / A100; `0x000000a6` on Drive A100 | `dual_rank[0]`, `dual_rank_bank[1]`, `SID_VAL[11]` |
| `0x009a03f0` | PLM candidate | `0xffffff8f` | survey only |
| `0x009a0470` | `FBPA_ECC_CTRL` | `0` with `MASTER_EN` read-only | see [ECC](../frontier/ecc.md) |
| `0x009a0554` | PLM candidate | `0xffffffcf` | survey only |
| `0x009a0838` / `0x009a083c` | `FBPA_VEND_ID_C0` / `C1` | `0x00000000` on all 15 cards | vendor ID is not exposed here |
| `0x009a0974` | `FBPA_TRAINING_STATUS` | `0x00000000` = FINISHED | SUBP0[1:0], SUBP1[3:2], value 2 = ERROR |
| `0x009a0bfc` | PLM candidate | `0x00000000` | survey only |
| `0x009a3cb4` / `b8` / `bc` | `I1500_INSTR` / `MODE` / `DATA` | `0x0000000f` / `0x00000008` / `0x40000000` | IEEE 1500 HBM test port |
| `0x009a3cc0` / `cc4` / `cc8` | `I1500_SHADOW_WIR` / `WDR` / `STATUS` | `0x000000f0` (RO) / per-die / `0x00000000` (idle) | `WDR` is `0x8000f000` on an 8 GB card, `0x8273ff83` on a 10 GB card |

### Per-FBPA unicast aperture

Instance *n* of every FBPA register lives at `0x00900000 + n*0x4000`, for n = 0..23:

| Register | Address | Notes |
|---|---|---|
| Per-FBPA `CFG0` | `0x00900200 + n*0x4000` | `0x07981800` on every live instance, both SKUs |
| Per-FBPA `CFG1` | `0x00900204 + n*0x4000` | must be hand-written only in a driverless context with no devinit |
| Per-FBPA `CSTATUS_RAMAMOUNT` | `0x0090020c + n*0x4000` | the verification target: `0x200` stock, `0x800` at the 40 GB tier, `0x1000` at the 64/80 GB tier |

> [!NOTE]
> **Stride is `0x4000`, not `0x400`**
>
> One adjudicated document carries a dropped-zero typo. The stride is `0x4000`, corroborated
> by the broadcast aperture being `0x009a0000`-`0x009a3fff`, i.e. exactly `0x4000` wide.

Per-FBPA capacity follows from LMR SCALE as `2^(SCALE+1)` MiB, which is precisely the value
CSTATUS_RAMAMOUNT reports. Cross-check: `0x800` x 20 FBPAs = 40960 MiB; `0x1000` x 16 FBPAs =
65536 MiB. A floorswept FBPA returns a `0xbadf20xx` sentinel instead.

**In the shipping driver path a single broadcast write is enough**, and the shipping driver
writes no per-FBPA register at all. In a driverless runtime with no devinit the broadcast alone
does not move CSTATUS and all 24 instances must be written by hand. Whether the broadcast is a
PRI priv-ring hardware mechanism or a software step was never directly instrumented.

### MMU / FB hub

| Address | Name | Value | Notes |
|---|---|---|---|
| `0x00100800` | `FBHUB_NUM_ACTIVE_LTCS` | `0x00000010` (16) on `10de:20c2`; `0x00000014` (20) on `10de:2082` | `0x14` also on A100 PCIe 40G/80G |
| `0x00100b10` | FB-geometry PLM | `0xffffff8f` | clean-room `FB_GEO_PLMS` entry |
| `0x00100b38` | FB-geometry PLM | `0xffffff8f` | six-PLM variant only |
| `0x00100b84` | PLM candidate | `0xffffff88` | survey only |
| `0x00100b90` | `FBHUB_MEM_PART_BCFG0` | `0x00000603` | matches documented init on every card |
| `0x00100b98` | `SYSMEM_HSHUB_CONNECTION_CFG` | `0x00000003` (BOTH, PCIe routing) | |
| `0x00100b9c` | PLM candidate | `0xffffffcf` | survey only |
| `0x00100ce0` | MMU local memory range | see above | |
| `0x00100ec0` | `MMU_NUM_ACTIVE_LTCS` | `0x05001414` on the 10 GB SKU and all three A100 SKUs; `0x04001410` reported on the 8 GB SKU | the per-SKU split is an open question, not a dissent; `...1410` is consistent with 16 LTCs, `...1414` with 20 |

### L2 / LTC

| Address | Name | Value | Notes |
|---|---|---|---|
| `0x0017e22c` | L2/LTC address-map register | `0x00280404` native | never programmed by anything, yet 40 GB works |
| `0x0017e2a0` / `0x0017e2a4` | per-LTC decode | | targeted by the clean-room v8 tool |
| `0x001402b4` | LTC companion | write of `0x00a00030` attempted | did not move the 40 GiB fold |

The clean-room v7-to-v8 change drove `DECODE_VAL` from `0x60000300` (2 GB per channel) to
`0x10000300` (4 GB per channel). On the 170HX the value stays at `0x70000300` throughout, which
remains unexplained. See [The 80 GB question](../frontier/80gb.md).

---

## Feature override and compute (`0x008238xx`)

This block is the compute unlock. Narrative in [Compute throttle](compute-throttle.md).

| Address | Name | Stock 170HX | Unlocked | PLM-gated | FLR | Touched by |
|---|---|---|---|---|---|---|
| `0x00823800` | `FEAT_OVR_ECC_PLM` | `0xffffff8f` (A100 SXM4 40G alone reads `0x0000abcf`) | `0xffffffff` when opened | is a PLM | unknown | Gen2 branch `xp3gTable`, never by master |
| `0x00823804` | **`FEAT_OVR_PLM`** | `0xffffff8f` | `0xffffffff` | is a PLM | **survives** (AON island) | shipping patch 0001, PLM index 3 |
| `0x00823808` | `FEAT_OVR_QUADRO` | **per-die and unexplained: see the note below** | | | | read only |
| `0x0082380c` | `FEAT_OVR_ECC` | `0x00888888` | | | | read only |
| `0x00823810` | `FEAT_OVR_ECC_1` | `0x002aaaaa` | | | | read only |
| `0x00823814` | `FEAT_READOUT_0` | `0x00000233` (RO); a reference GA100 board read `0xef8ff100` | | RO | | field layout undocumented |
| `0x00823818` | **`FEAT_READOUT_1`** | `0x016db6ed` | **`0x00000000`** | RO | | the cleanest "is this card unlocked" test |
| `0x0082381c` | **`FEAT_OVR_SM_SPEED_SELECT` (SS0)** | varies: `0x53540175`, `0x12103060` | `0x88888888` | yes, via `0x00823804` | **survives** | shipping patch 0001 |
| `0x00823820` | **`FEAT_OVR_SM_SPEED_SELECT_1` (SS1)** | varies: `0x00000000`, `0x00000003` | `0x00000008` | yes | **survives** | shipping patch 0001 |
| `0x00823824` | `FEAT_OVR_ROW_REMAP` | `0x00000000` on both 170HX SKUs | | | | read only |
| `0x00823828` | `FEAT_READOUT_2` | `0x00000000` on 170HX; `0x00000007` on all A100 and Drive parts | | RO | | read only |
| `0x0082382c` | `FEAT_READOUT_2` (alias in one dump) | `0x0000000a` | | RO | | naming unsettled between two dumps |
| `0x00823b00` | row-remapper PLM (`FEAT2`) | `0xffffff8f` | `0xffffffff` when opened | is a PLM | one sweep read it open after FLR | Gen2-family patch 0001 only |

> [!NOTE]
> **Open problem: `0x00823808` `FEAT_OVR_QUADRO` reads differently in every dump**
>
> Per-die and unexplained. Observed: `0x00100183` (stock, PLM range scan, medium),
> `0x00000081` (post-unlock probe, medium), `0x00000181` / `0x00000182` (two physical 170HX
> units, high, one of the 13 binning differences), `0x01000282` (A100 80 GB). Read only.
> **Open question:** why the value differs across all three dumps; something in the unlock or
> the driver may be touching the Quadro-versus-consumer classification word, which could be the
> lever for driver-visible feature classes. Next step: re-read this register before and after
> each stage of the shipping sequence on one card.

**What SS0/SS1 encode.** `0x0082381c` holds eight 4-bit fields for IMLA0-3, FMLA16, FMLA32,
FFMA and DP; `0x00823820` holds the ninth field for IMLA4. Each nibble reads best as
`[enable | 3-bit speed]`, so `0x8` means "override enabled, speed 0 = full rate". `0x88888888`
therefore says "override enabled, full rate" on all eight SS0 units, and `0x00000008` does the
same for IMLA4. No stock dump anywhere in the archive has any nibble at or above `8`. This
encoding is inferred, not documented.

> [!CAUTION]
> **Do not use SS0/SS1 readbacks as a per-part reference**
>
> They are runtime state, not fuse state. Two archived dumps of the same A100 80 GB device ID
> disagree (`0x00112011`/`0x00000002` versus `0x00343015`/`0x00000004`). Use `0x00823818 == 0`
> instead.

Both writes are required; writing only one is not enough. Both are identical on both SKUs and
in every one of the twelve unreleased branches: no branch ever experimented with different
compute values.

---

## Fuses and OTP shadows (`0x0082xxxx`)

These are read-only fuse-sense reflections unless noted. See
[Fuses and OTP](../hardware/fuses-and-otp.md).

### Fuse control

| Address | Name | 170HX | Notes |
|---|---|---|---|
| `0x00820000` | `FUSE_FUSECTRL` | `0xe0040000` | identical on all 15 cards in the cohort |
| `0x00820040` | `FUSE_EN_SW_OVERRIDE` | `0x00000000` | `0x00000001` on consumer and engineering-sample parts; writable and persistent on the 170HX but changes nothing observable |
| `0x00820078` | `FUSE_EN_PROGRAM` | `0x00000001` | all 15 cards |
| `0x0082007c` | `FUSE_DIS_PROGRAM` | `0x00000000` | `0xbadf5040` on GA10x |
| `0x00820080` | `FUSE_BYPASS_STATUS` | `0x00000000` | `0xbadf5040` on GA10x |
| `0x00820084` | `FUSE_DIS_SW_OVR` | `0x00000001` | all 15 cards; HS writes bounce |
| `0x0082038c` | `FUSE_QUADRO_WR_SEC` (`OPT_SECURE_FEATURE_OVERRIDE_QUADRO_WR_SECURE`) | `0x00000001` | permits `0x00823804` to be opened at all |
| `0x008203f0` | **`FUSE_FEAT_OVR_DIS` (`OPT_FEATURE_FUSES_OVERRIDE_DISABLE`)** | `0x00000000` | **the master kill fuse, unblown. This single zero is why the entire unlock exists.** |
| `0x008203f4` | `OPT_INTERNAL_SKU` | `0` | |
| `0x0082074c` | `FUSE_OPT_SECURE_GSP` | `0x00000001` | all 15 cards |
| `0x00820618` | `FUSE_FBPA_MEM_WR_SEC` | `0x00000001` | all 15 cards |
| `0x00821060` | `OPT_SKU_ID` | `0x00000068` (10 GB, `0x2082`); `0x00000080` (8 GB, `0x20C2`) | both physical probe units were 10 GB and read `0x68`; the `0x80` value comes from a clean-room silicon read of an 8 GB card |

### SM speed-select fuses (the throttle itself)

All read `0x00000005` on the 170HX (3-bit field, 0 = full rate, 5 = divide-by-32) and
`0x00000000` on A100 SXM4 40 GB, A100 PCIe 40/80 GB, A10, A5000, A6000, Drive A100 and the
96-SM `0x20bb` DRIVE-PG199-PROD part. **They are unchanged by the unlock**: the override
supersedes them.

| Address | Name | 170HX | Notes |
|---|---|---|---|
| `0x008200fc` | `FUSE_SS_PLM` / `OPT_PLM` | `0xffffffff` in one sweep, `0x000003ff` in another | two names for one register: `OPT_PLM` is the branch-code label, `FUSE_SS_PLM` the clean-room tooling name. Not written by master |
| `0x00820224` | `FUSE_SS_DP` | `0x00000001` | separate 1-bit fuse (0 full, 1 reduced) |
| `0x0082059c` | `FUSE_SS_FFMA` | `0x00000005` | |
| `0x008207d4` | `FUSE_SS_FMLA16` | `0x00000005` | |
| `0x008207d8` | `FUSE_SS_FMLA32` | `0x00000005` | RTX 3070 reads 1 |
| `0x008207dc` | `FUSE_SS_IMLA0` | `0x00000005` | |
| `0x008207e0` | `FUSE_SS_IMLA1` | `0x00000005` | |
| `0x008207e4` | `FUSE_SS_IMLA2` | `0x00000005` | |
| `0x008207e8` | `FUSE_SS_IMLA3` | `0x00000005` | |
| `0x008207ec` | `FUSE_SS_IMLA4` | `0x00000005` | RTX 3070 reads 1 |

### PCIe fuses

| Address | Name | 170HX | Every other Ampere part probed | Notes |
|---|---|---|---|---|
| `0x0082057c` | `FUSE_PCIE_GEN23_DIS` (`OPT_PCIE_BOOT_GEN23_DISABLE`) | `0x00000001` | `0x00000000` | **hard read-only.** Attempted from host, HS ROP and the Booter payload; always fails with `rd=0x00000001`. Gen2 works anyway |
| `0x00820580` | `FUSE_PCIE_GEN3_DIS` (`OPT_PCIE_BOOT_GEN3_DISABLE`) | `0x00000001` | `0x00000000` | |
| `0x00820520` | `FUSE_PCIE_MAGIC_D` | `0x16680000` (bit 25 set = `GEN4_SPEED_DISABLED`, NVIDIA bug 2220334) | `0x00200000` on A100 and Drive GA100 | writability disputed |
| `0x00820584` | `FUSE_DEVID_SW_OVR_DIS` | `0x00000001` | `0x00000001` | |
| `0x00820394` | `OPT_PCIE_LANE_DISABLE` | `0x00000000` | `0x00000000` | proof the x4 width is board-level, not fused |
| `0x0082082c` | `CTRL_OPT_PCIE_LANE` | `0x00000000` | `0x00000000` | |
| `0x00820c2c` | `STATUS_OPT_PCIE_LANE` | `0x00000000` | `0x00000000` | |
| `0x008204d8` | `OPT_PCIE_DEVIDA` | `0x000020c2` (8 GB), `0x00002082` (10 GB) | A100 `0x20b2` | SKU-id fuse |
| `0x0082056c` | `OPT_PCIE_DEVIDB` | `0x000020c2` on both SKUs | A100 `0x20f2`; PG199 `0x000020fb` | on the 10 GB card DEVIDA and DEVIDB disagree |
| `0x00820148` | OTP spare bit | `0x00000000` | | never settable |

> [!NOTE]
> **Open problem: is `0x00820520` writable?**
>
> One analysis annotates bit 25 "(writable)"; one clean-room tool writes `0x00200000` to it as
> part of a working Gen2 chain; the PCIe field manual lists it read-only; and the shipping
> Gen2 patch only ever reads it. Nobody has published a write-then-readback. Gen4 is untestable
> anyway because no one working on it has a Gen4 host. See
> [Gen3 and Gen4](../frontier/pcie-gen3-gen4.md).

### NVLink fuses

| Address | Name | 170HX | Notes |
|---|---|---|---|
| `0x00820684` | `FUSE_NVLINK_DIS` (`OPT_NVLINK_DISABLE`) | `0x00000007` (all three bits of [2:0]) | `0x00000000` on A100 SXM4 40G / PCIe 40G / PCIe 80G / A10 / A5000 / A6000 / RTX 3090 / 3090 Ti; `0x00000001` on RTX 3080 / 3080 Ti; `0x00000007` on Drive A100 too |
| `0x00820db8` | `STATUS_OPT_NVLINK` | `0x00000007` (RO mirror) | shared with the Drive A100 |
| `0x008209b8` | `CTRL_OPT_NVLINK` | `0x00000000` (bits 15:0, per link) | override shadow reads zero on every card probed |
| `0x00820820` | `CTRL_OPT_PERLINK` | | never write-tested |

No `FEAT_OVR` entry exists for NVLink anywhere in the `0x00823800`-`0x0082382c` block, and no
branch contains NVLink code. See [NVLink](../frontier/nvlink.md).

### Floorsweep fuses

Per-card, not per-SKU. Every surveyed 170HX lands at 70 SM regardless of which GPCs are off.

| Address | Name | Observed | Notes |
|---|---|---|---|
| `0x00820350` | `OPT_GPC_DISABLE` | `0x85`, `0x45`, `0x13`, `0xa8` on four different cards | HS writes bounce, value is latched |
| `0x00820364` | `OPT_FBP_DISABLE` | `0x00000840` (10 GB card), `0x00000852` (community dump), `0x00000009` / `0x00000180` (two units) | |
| `0x00820368` | `OPT_FBPA_DISABLE` | `0x000000c3` (10 GB card: FBPA 00/01/06/07 off, 20 live), `0x00c0330c` (8 GB card: FBPA 02/03/08/09/12/13/22/23 off, 16 live) | |
| `0x0082036c` | `OPT_FBIO_DISABLE` | mirrors `0x00820368` | |
| `0x008202c4` | `OPT_ROP_L2_DISABLE` | mirrors `0x00820368` | |
| `0x00820398` | `OPT_SPARE_FS` | `0x00000000` | |
| `0x008203a4` | `OPT_SYS_PIPE_DISABLE` (inferred) | `0xfe` on both 170HX, `0x0` on GA106 | inferred syspipe mask; if the mapping is correct, bits 1–7 disable syspipes 1–7; host BAR0 writes do not latch; matches the single `1g.64gb` MIG profile |
| `0x008205c4` | `OPT_GPC_DEFECTIVE` | `0x00000000` on several cards whose DISABLE had three bits set; `0x81` on one 10 GB card | "disabled" and "defective" are separate masks: some disabled GPCs are physically good silicon |
| `0x008205cc` | `OPT_FBP_DEFECTIVE` | `0x00000840` (10 GB card) | |
| `0x008205d0` / `0x008205d4` / `0x008205e8` | `OPT_FBPA_DEFECTIVE` / `FBIO_DEFECTIVE` / `ROP_L2_DEFECTIVE` | `0x00c03000` each | |
| `0x00820818` | `CTRL_OPT_FBPA` | `0x00000000` | no override present |
| `0x00820838 + i*4` | `FUSE_CTRL_OPT_TPC_GPC(i)` | `0x00000000` | **remove-only (subtractive)**: writing it never adds a TPC back |
| `0x00820938` | `CTRL_OPT_FBP` | `0x00000000` | |
| `0x00820840` | MIG enable | `0` stock; setting bit 0 enables MIG and was reported persistent | **not in the shipping tree**; a repo-wide grep for `0x820840` returns nothing |
| `0x00820c00` | `STATUS_HALF_FBPA` | `0` | no half-capacity fuses to recover |
| `0x00820c14` | `STATUS_OPT_FBIO` | `0x00c0330c` (8 GB card) | |
| `0x00820c18` | `STATUS_OPT_FBPA` | `0x00c0330c` / `0x000000c3` | **this is the correct address; `0x00820c14` is FBIO** |
| `0x00820c1c` | `STATUS_OPT_GPC` | always mirrors `0x00820350` | |
| `0x00820c38 + i*4` | `FUSE_STATUS_OPT_TPC_GPC(i)` | GPC0/3/5 = `0xff`, others = `0x01` on one card | |
| `0x00820d38` | `STATUS_FBP` | `0x00000180` on one unit | |
| `0x00820e30` | syspipe mask, possible applied copy (inferred) | `0xfe` on 170HX, `0xbadf5040` on GA106 | same value as `0x008203a4`; host BAR0 write-probe `0xfc` did not latch — not a writable shadow |

### Topology scalars (`0x0002xxxx`)

Read-only, and they describe the full GA100 die, not the harvested part.

| Address | Name | 170HX |
|---|---|---|
| `0x0002241c` | `NV_PTOP_FS4` | `0x00000000` on the 8 GB card (`0x20c2`); `0x00000081` on the 10 GB card (`0x2082`), and also on A100 80 GB, RTX 3070, GA10x and Drive `0x20bb`. Bit 0 is `GEN2_PCIE`, bit 7 is `GEN2_PCIE_SPEED` |
| `0x00022430` | `PTOP_SCAL_NUM_GPCS` | `0x00000008` (GA10x reads 7) |
| `0x00022434` | `PTOP_SCAL_TPC_PER_GPC` | `0x00000008` |
| `0x00022454` | `PTOP_SCAL_NUM_LTCS` | `0x00000018` (24) |
| `0x00022458` | `PTOP_SCAL_FBPA_PER_FBP` | `0x00000002` (RTX 3090 reads 1) |
| `0x0002246c` | `PTOP_SCAL_NUM_NVLINK` | `0x0000000c` (12); GA10x reads 4 |
| `0x00022470` | `PTOP_FS_STATUS` | `0x0000003f`; bit0 TPC, bit1 GPC, bit2 FBP, bit3 ROP, bit4 FBIO |
| `0x00120078` | `RING_ENUM_GPC` | `5` on every 170HX; never moved by any write attempt |
| `0x00001404` | `PBUS_SW_SCRATCH(1)` | `0x20042000`, bit 14 = 0 on all cards surveyed |
| `0x00000000` | `PMC_BOOT_0` | `0x170000a1` on every valid GA100; GA10x control reads `0xb74000a1` |
| `0x008204bc` | `OPT_SLT_REV` | read by `ga100_topology_report.py` |

---

## PCIe: XVE, XP3G and XP-PL

Everything in this section is **unreleased-branch material**. Shipping `master` contains
patches `0001` through `0006` only, has no `pcie:` block in `constants.yaml`, and a
case-insensitive grep of its installer, remover, README and build script for
`gen2|gen 2|pcie|iommu|retrain|RMPcieLinkSpeed` returns zero hits. See
[PCIe Gen2](pcie-gen2.md) and [PCIe subsystem](../hardware/pcie-subsystem.md).

> [!WARNING]
> **Experimental**
>
> Patch `0007-pcie-gen2.patch` exists on branches `debug-gen2`, `Gen2`, `far` and `deced`;
> `0008-pcie-gen2-probe-retrain.patch` on `Gen2`, `far` and `deced`. Nothing here has been
> merged to master.

### XVE config-space shadow (BAR0 base `0x88000`)

Config reads come fresh from this shadow on every access, which is why rewriting it at runtime
and then forcing a retrain makes the host re-read corrected capabilities. The PCIe Express
capability sits at config offset `0x78`, so config `0x78 + X` maps to BAR0 `0x88078 + X`.

| Address | Config offset | Name | Stock | After 0007 | Notes |
|---|---|---|---|---|---|
| `0x00088084` | CAP_EXP+0x0c | `LINK_CAP` (LnkCap) | `0x00456101` | `0x00456102` | bit 20 (DLL Link Active Reporting Capable) is **clear**, which breaks 0008's success test |
| `0x00088088` | CAP_EXP+0x10 | `LINK_CTRL_STATUS` | LnkSta `0x1041` | `0x1042` | `PCIE_LINK_SPEED_OF(stat) = ((stat) >> 16) & 0xF` |
| `0x000880a4` | CAP_EXP+0x2c | `LINK_CAP2` (LnkCap2) | `0x00000002` (2.5 GT/s only) | `0x00000006` (G1/G2) | hardware read-only, marked `R-EVF`: `setpci` writes are silently dropped |
| `0x000880a8` | CAP_EXP+0x30 | `LINK_CTRL_2` (LnkCtl2) | `0x0000` / register reads `0x00000001` | bits[3:0] = `0x2`, bits[19:16] = `0xF` | A100 reads `0x001f0004` here |
| `0x0008841c` | | `PRIV_MISC_1` | `0x20340500` | `0x20342d00` | set bits 11 and 13, clear bits 12 and 14; **succeeds on the first attempt and survives BooterLoad** |
| `0x0008860c` | | `VSEC_DEVICE` | `0x00000800` | wants `0x00000801` | **write FAILS** twice on silicon; readback stays `0x00000800` |
| `0x00088610` | | `VSEC_HIERARCHY` | `0x00001001` | clear bit 12, set bit 0 | plain host write after the Booter phase |
| `0x0008872c` | | LTSSM / `XVE_OVR` | | `0x00000006` | the patch's own log calls this "skip mid-boot retrain". Values `0x2` and `0xa` expose extra Gen2 behaviour under VFIO but eventually wedge the QEMU function |
| `0x00088ab4` | | `XVE_B` PLM | | `0xffffffff` | Gen2-family PLM table |
| `0x00088ce4` | | unnamed | `0x0000003f` | | A100 reads `0x00000014` |
| `0x00088fe8` / `0x00088fec` / `0x00088ff0` | | `XVE_D0` / `D4` / `D8` PLMs | | `0xffffffff` | `xp3gTable` entries |
| `0x00088ff4` | | `XVE` PLM | | `0xffffffff` | Gen2-family PLM table |
| `0x00088ff8` | | `XVE_C` PLM | | `0xffffffff` | Gen2-family PLM table |

### XP3G PHY-rate override block (`0x0008e1xx` / `0x0008e1xx`)

| Address | Name | Value written by 0007 | Notes |
|---|---|---|---|
| `0x0008e100` | `XP3G_STATUS` base | (read) | |
| `0x0008e110` | `XP3G_OVR0` | `0x00000001` | observed reading back `0x00000004` in an earlier standalone probe |
| `0x0008e11c` | `XP3G_OVR3` | `0x00000004` | |
| `0x0008e120` | `XP3G_VAL0` | `0x00000000` | |
| `0x0008e12c` | `XP3G_VAL3` | `0x00200000` | |
| `0x0008e1b0` | `XP3G_PLM` | `0xffffffff` | opens fine; reads back `0xffffffff` |
| `0x0008e1b4` | `XP3G_PLM4` | `0xffffffff` | |
| `0x0008e1b8` | `XP3G_PLM8` | `0xffffffff` | |
| `0x0008e1bc` | `XP3G_PLMC` | `0xffffffff` | |

An isolated XP3G override with the PLM open drove the rate field to a Gen3-capable
`0x00340036` and the link still trained at Gen1. That refuted XP3G as a standalone lever, but
it is one component of the combination that later worked.

### XP-PL link-config block (`0x0008cxxx`)

Written as **plain host BAR0 writes**, after the Booter phase, with no privilege escalation:

| Address | Name | Operation | Notes |
|---|---|---|---|
| `0x0008c040` | `LINK_CONFIG_0` | bits[19:18] `MAX_RATE` = `0x2` | read-modify-write: clear mask `0x000c0000`, then OR in `2 << 18` |
| `0x0008c044` / `0x0008c048` / `0x0008c04c` | LINK_CONFIG cluster | (rejected HS writes) | a *different* cluster from the three that work |
| `0x0008c080` | link WIDTH | A100 reads `0x00001010` | |
| `0x0008c1c0` | `PL_LINK_RATE` | `= 0x00240036` | A100 reads `0x00040036` |
| `0x0008c2c0` | `CYA_0` | clear bit 2 (`DIS_G2`) | the central lever |

### OPTB PLM block written by 0007

Ten registers, `0x008200d0`, `d4`, `d8`, `dc`, `e0`, `e4`, `e8`, `ec`, `f0`, `f4`, all set to
`0xffffffff`, plus `0x00823800` `FEAT_OVR_ECC_PLM` and `0x0082057c` `OPT_GEN23` (attempted
`0x00000000`, always fails).

### Write-count accounting for `0007`

`xp3gTable` has **23** entries: 18 PLM opens plus 5 value writes. `VSEC_DEVICE 0x0008860c` and
`PRIV_MISC_1 0x0008841c` are handled outside the table, giving **25 Booter-routed writes**.
Each gets two attempts, with `0x001fa824` / `0x001fa828` re-armed before each. Then come six
plain host writes: `0x00088610`, `0x000880a8`, `0x0008c2c0`, `0x0008c040`, `0x0008c1c0` and
`0x0008872c`.

### Registers that are PROT-walled or poisoned from the injection point

| Address | Behaviour |
|---|---|
| `0x00088070`, `0x0008808c`, `0x00088090` | reads return 0, writes ignored |
| `0x00085080`, `0x00085084` | read `0xbadf1100`; "GSP writes `0x85084`" is true, but at a privilege the injection point never reaches |
| `0x00409664`, `0x00409668` | `0xbadf5040` on every Ampere card, including unthrottled ones |

> [!NOTE]
> **Do not repeat `0x8808c` as the LnkCap2 mirror**
>
> One field manual lists `NV_XVE_LINK_CAPABILITIES_2` as "cfg 0xA4 / BAR0 mirror `0x8808c`",
> which is internally inconsistent. With the XVE mirror base at `0x88000`, config `0xA4` maps
> to `0x880a4`, and that is what both patch 0007 and the independent `pcielink.sh` use.

---

## Graphics, SKED and FECS: investigated, never used

Nothing in the shipping tree touches any of these. A repository-wide grep for `0x504204`,
`0x8200fc`, `0x82038c`, `0x8203f0`, `0x823818`, `0x820224`, `0x82059c` and `0x820840` returns
zero hits.

| Address | Name | 170HX | Verdict |
|---|---|---|---|
| `0x00407000` | `SKED_HW_BLK` | `0x00004042` (`0xbadf1201` pre-driver) | |
| `0x00407010` | `SKED_PM_UNK10` | `0x00000000` | |
| `0x00407020` | `SKED_TRAP` | `0x00000000` | |
| `0x00407024` | `SKED_TRAP_EN` | `0x3dfffffc`, identical to A100; RTX 3090 reads `0xbdfffffc` (bit 31 only) | |
| `0x00407054` | `SKED_UNK54` | `0x60000600` (pre-driver) or `0x600000c0`; **0 on both A100 and RTX 3090** | the most-referenced undocumented SKED register in the GSP firmware (13 references) and the only register in the 13-card cohort that is non-zero on the 170HX and zero on the controls. Never write-tested. The driver clears it during GR init |
| `0x00408970` | `gpcMask` | `0xdc`, re-asserts after every forcing attempt | dead end |
| `0x00409664` | `FECS_FEAT_OVERRIDE` | `0xbadf5040` | read-blocked on every Ampere card, so the value carries no information |
| `0x00409668` | `FECS_FEAT_READOUT_1` | `0xbadf5040` | same |
| `0x00504204` | `SM_ISSUE_RATE_MODIFIER` | `0x00000005` with a driver, `0xbadf1201` without | **not** the throttle: reads `0x00000005` on 13 comparison Ampere cards and on a 96-SM `0x20bb` GA100 whose every speed-select fuse is 0. Host-writable; zeroing it changes nothing |

> [!NOTE]
> **Open problem: does `0x00504204` impose a residual limit on an already-unlocked card?**
>
> Nobody has run the obvious A/B: write it to 0 on an unlocked card and re-run the benchmark
> suite. The write primitive already exists.

---

## The payload offset table

The Booter's LS-signature verification (`booterVerifyLsSignatures_TU10X` at IMEM `0x29c4`)
performs an unbounded DMA whose length is taken straight from `sizeOfSignature` in `WprMeta`.
The driver sets that to `SEC2_POSTBL_TIMING_SIGNATURE_SIZE = 0x0000f800` (63,488 bytes) and the
DMA destination is DMEM `0x0800`, so the payload maps 1:1 onto DMEM `0x0800`..`0xffff`
(`0x0800 + 0xf800 = 0x10000`, exactly the top of the 64 KB DMEM). **DMEM address = payload
offset + `0x800`.**

Every dword in the buffer is first filled with `SEC2_POSTBL_TIMING_FILL_DWORD = 0x000004a7`
(15,872 dwords), then exactly 24 slots are overwritten:

| Payload offset | DMEM | Value | Role |
|---|---|---|---|
| all | `0x0800`-`0xffff` | `0x000004a7` | background fill dword |
| `0x1100` | `0x1900` | `0x00000007` | purpose unidentified |
| `0x5b40` | `0x6340` | `0xc0deca7e` | **fake canary written into the stack-guard global** |
| `0xf754` | `0xff54` | *writeValue* | the value argument, the lowest tail slot |
| `0xf758` | `0xff58` | `0xc0deca7e` | saved-canary slot |
| `0xf75c` | `0xff5c` | `0x00000cbd` | |
| `0xf76c` | `0xff6c` | *writeAddr* | the address argument |
| `0xf774` | `0xff74` | `0x00001fbd` | |
| `0xf780` | `0xff80` | `0x00000000` | |
| `0xf788` | `0xff88` | `0x000010aa` | **the BAR0-master write gadget (`reg_write_indirect`)** |
| `0xf78c` | `0xff8c` | `0x0000815a` | |
| `0xf790` | `0xff90` | `0x00008e18` | |
| `0xf794` | `0xff94` | `0xc0deca7e` | saved-canary slot |
| `0xf798` | `0xff98` | `0x0000815a` | |
| `0xf79c` | `0xff9c` | `0x00000000` | |
| `0xf7a0` | `0xffa0` | `0xc0deca7e` | saved-canary slot |
| `0xf7a4` | `0xffa4` | `0x00001fbd` | |
| `0xf7b0` | `0xffb0` | `0x0000ffbc` | |
| `0xf7b8` | `0xffb8` | `0x0000582d` | |
| `0xf7c4` | `0xffc4` | `0xc0deca7e` | saved-canary slot |
| `0xf7c8` | `0xffc8` | `0x00000cbd` | |
| `0xf7d8` | `0xffd8` | `0x00000003` | |
| `0xf7e0` | `0xffe0` | `0x00001fbd` | |
| `0xf7f4` | `0xfff4` | `0x00000ccb` | see the open problem below |
| `0xf7f8` | `0xfff8` | `0x00007f2f` | outermost slot |

This table is **byte-identical in shipping `master` and in all twelve archived branches**
(verified by checksum and by a grep of `0xc0deca7eU`, which occurs exactly five times in every
copy).

**Load-bearing DMEM addresses referenced by the table:**

| DMEM | Meaning |
|---|---|
| `0x0100` and below | nothing is allocated here, which killed the "mega-ROP staged in low DMEM" idea |
| `0x0530` | DMA/engine config descriptor |
| `0x0600` | `WprMeta`, a 256-byte structure |
| `0x06fc` | where the Booter stores `0xa0a0a0a0` on the `r4 == 0` branch at IMEM `0x27fa`; **unrelated** to the `0x1fa824`/`0x1fa828` WPR2 registers |
| `0x0800` | base of the DMA'd signature buffer, i.e. payload offset 0 |
| `0x103c` onward | crypto-session descriptor |
| `0x2383`, `0x8e08` | register descriptor tables, which the payload linearly smashes |
| `0x6340` | **the stack-canary global**, 25408 decimal |
| `0x8700` | end of booter code/data |
| `0xffec` | the slot that feeds `main`'s exit status and decides whether `secure_teardown` runs |

> [!NOTE]
> **Open problem: `0x00000ccb` at DMEM `0xfff4`**
>
> `0x0ccb` is `regtable_rw_indexed`, which indexes the very descriptor tables at DMEM `0x2383`
> and `0x8e08` that the payload smashes, and a 2026-07-06 isolation matrix showed every
> write-carrying rejoin chain dying at `0xccb`. Yet the shipping payload plants `0x00000ccb`
> at `0xfff4` and the unlock demonstrably works. Settled by tracing whether `0xfff4` is ever
> loaded into PC during the unwind or is only a live-through saved slot in a frame that is
> never returned through.

> [!NOTE]
> **Open problem: the unexplained payload constants**
>
> `0x00000007` at DMEM `0x1900`, `0x00000003` at `0xffd8`, `0x0000582d`, `0x0000ffbc`,
> `0x00008e18`, `0x0000815a` (twice), `0x00000cbd` (twice), `0x00001fbd` (three times),
> `0x00007f2f`, and the fill dword `0x000004a7` itself. The ROP write-ups name a neighbouring
> gadget family (`0x1fb9`, `0x1fca`, `0x814e`, `0x8173`, `0x7f82`), so these are plausibly the
> same tail translated. One pass over the annotated disassembly should resolve all of them.

### Payload override hook

`SEC2_POSTBL_TIMING_DMEM_PATH = "/lib/firmware/nvidia/ga100/gsp/dmem.bin"` is read into the
freshly created `0xf800` buffer, falling back to the built-in template (pre-seeded with
`writeAddr 0x009a0148`, `writeValue 0xffffffff`) if absent. Absence is reported as status
`0x59` and is benign.

### Payload variants across the archive

| Variant | Size | DMA base | Canary value | Canary slots |
|---|---|---|---|---|
| Shipping `master` and all 12 branches | `0xf800` = 63,488 B | `0x0800` | `0xc0deca7e` | `0x6340`, `0xff58`, `0xff94`, `0xffa0`, `0xffc4` |
| Clean-room ROP write-ups (2026-07-07/08/13) | `0xf800` | `0x0800` | `0xfaceb13d` | `0x6340`, `0xff58`, `0xff94`, `0xffdc`, `0xfff4` |
| Superseded `builder.py` / `patcher.py` | `0xf700` = 63,232 B | `0x0900` | `0xdead2c20` at `0x2c20` | production-image offsets, none reused |

The canary **value** is arbitrary as long as it is uniform across the guard global and every
saved copy; the **address `0x6340` is the load-bearing fact**. A one-off message giving `0x6440`
is a slip: `0x5b40 + 0x900 = 0x6440` is what you get from the older documented DMA base of
`0x0900`.

---

## Numbers that are not BAR0 addresses

Every entry here has been mistaken for a register address at least once in the archive.

| Number | What it actually is |
|---|---|
| `0x02449000`, `0x02669000`, `0x02779000` | FBPA CFG1 **values** (stock, 40 GB tier, 64/80 GB tier). The nibble-shifted spellings `0x24490000`, `0x26690000`, `0x27790000` that circulated in chat are transcription slips |
| `0x00000208`, `0x0000020B`, `0x00000288`, `0x0000028A`, `0x0000028B` | MMU LMR **values** |
| `0x0000001000000000`, `0x0000000A00000000`, `0x0000001400000000`, `0x0000000200000000` | 64 GiB, 40 GiB, 80 GiB and 8 GiB byte counts (`targetFbBytes` / `fb_length` / `stockFbBytes`) |
| `0x88888888`, `0x00000008` | SS0 and SS1 **values** |
| `0x0000f800`, `0x000004a7`, `0xc0deca7e`, `0xfaceb13d` | payload size, fill dword, canary values |
| `0x000010aa`, `0x000010b9`, `0x00001196`, `0x00001064`, `0x00008224`, `0x00008264`, `0x00008262`, `0x00007f82`, `0x0000814e`, `0x00008137`, `0x00008173`, `0x00008117`, `0x00008119`, `0x0000810d`, `0x00000ccb`, `0x00001fbd`, `0x00007f2f` | **Falcon IMEM addresses inside the Booter**, not BAR0 offsets. `0x10aa` is `reg_write_indirect`; entering at `0x10b9` skips the `r10`/`r11` copies |
| `0x0001c000`, `0x0001c100`, `0x0001c200`, `0x0001c300`, `0x00009100`, `0x00012000` | Falcon CSB space (`I[...]`), unreachable from the host |
| `0x00200000` | the A100/Drive value of `FUSE_PCIE_MAGIC_D`, and separately the value written to `XP3G_VAL3` |
| `0x00240036`, `0x00340036` | `PL_LINK_RATE` values (the Gen2 one that ships, and the Gen3-capable one that trained nothing) |
| `0x1ffffe00` | the WPR2_LO teardown **value** |
| `0x800000f1`, `0x800000f2` | BAR0-master read and write **command words** |
| `0x1312d00` | the BAR0-master watchdog seed, 20,000,000 decimal |
| `0x0000abcf`, `0x0004cb8f`, `0xffffff8f`, `0xfffffe8e`, `0xffffffcf`, `0xffffff88`, `0xfffff0ff` | PLM **contents** |
| `0x170000a1` | the `PMC_BOOT_0` value that identifies GA100 silicon |

---

## Cross reference: which patch or tool touches which register

| Patch / tool | Registers written | Registers read |
|---|---|---|
| `0001-sec2-postbl-plm-ss-cfg.patch` (master) | `0x001fa7cc`, `0x009a0148`, `0x001fa7c4`, `0x00823804` (PLMs); `0x001fa824`, `0x001fa828` (re-arm); `0x0082381c`, `0x00823820`, `0x009a0204`, `0x00100ce0` (unlock) | all of the above, plus GSP static-info `fb_length` and the last FB region's `limit`, `reserved`, `supportCompressed`, `supportISO`, `performance` (= 20) |
| `0002-booter-verify.patch` | none | `0x00823804`, `0x0082381c`, `0x00823820`, `0x009a0204`, `0x00100ce0` (the project's own canonical five-register verify line) |
| `0003-late-pma.patch` | none | uses the `0x200000000` (8 GiB) split point between the stock region and the late-PMA extension, **for both SKUs**, including the 10 GB card whose true stock size is `0x280000000` |
| `0004-bar0-pramin-clamp.patch` | none | clamps the PRAMIN window to `(0x2000ULL << 20) - DRF_SIZE(NV_PRAMIN)` when `Ram.fbAddrSpaceSizeMb > 0x2000` on `0x20C2` and `0x2082` |
| `0005-ce-scrub-workarounds.patch` | none | forces `NV_MMU_PTE_KIND_GENERIC_MEMORY` and disables virtual-mode CE scrubbing on both device IDs |
| `0006-persistent-sw-state.patch` | none | sets `NV_FLAG_PERSISTENT_SW_STATE` for `0x20C2` and `0x2082` |
| `0007-pcie-gen2.patch` (branch) | the 23-entry `xp3gTable`, plus `0x0008860c`, `0x0008841c`, `0x00088610`, `0x000880a8`, `0x0008c2c0`, `0x0008c040`, `0x0008c1c0`, `0x0008872c` | `0x00088084`, `0x000880a4`, `0x00088088`, `0x0082057c`, `0x00820580`, `0x00820520`, `0x0008e1b0` |
| `0008-pcie-gen2-probe-retrain.patch` (branch) | BAR0 `0x8c2c0`, `0x8c040`, `0x8872c`; PCIe capability `LNKCTL2` on GPU and bridge; `LNKCTL` Retrain Link on the bridge only | GPU `LNKSTA`, polled 20 times at 100 ms |
| Branch `80` patch `0001` | identical to master except `cfg1Value = 0x02779000U` and `targetFbBytes = 0x0000001400000000ULL` on the 10 GB path | |
| Gen2-family patch `0001` | master's four PLMs plus `0x00088ff4`, `0x00088ab4`, `0x00088ff8`, `0x00823b00`, `0x008200fc` | |
| `probe.sh` / `ga100_topology_report.py` | nothing | `0x00000000`, `0x00820350`, `0x00820c1c`, `0x008205c4`, `0x00120078`, `0x00022430`, `0x00001404`, `0x00118f78`, `0x008204d8`, `0x008204bc`, per-GPC `OPT_DISABLE` / `RECONFIG` / `CTRL_OPT` / `STATUS` / `RECONF_OVR`; `FBPA_BASE = 0x900000`, `FBPA_STRIDE = 0x4000` |
| `nuke.sh` | three-write ROP per cycle, all targets `0xffffffff` | the 26-PLM candidate set |
| `refire_chain_v2/v6/v9.py` | `0x001fa824` (teardown `0x1ffffe00`), the FB-geo PLM set, `0x009a0204`, `0x00820520` in one variant | `0x009a0204`, `0x008403c4` as the readiness gate |
| `geo_flr_survival_map_20260716.sh`, `plm_flr_survival_20260716.sh`, `fire_vram_featovr_sweep.sh` | opens PLMs, then FLRs | the whole 26-register PLM survey |

---

## The FLR survival table

This is the asymmetry that shaped the entire project: compute shipped weeks before memory
because compute state is in the always-on island and memory state is not.

| Register | Survives FLR? | Evidence |
|---|---|---|
| `0x0082381c` (SS0) | **yes** | measured before/after on a 10 GB card |
| `0x00823820` (SS1) | **yes** | same |
| `0x00823804` (`FEAT_OVR_PLM`) | **yes** | the only PLM in the 26-register survey marked AON |
| `0x00823b00` (row-remapper PLM) | probably | one in-HS sweep read `0xffffffff` after FLR, but opening it did not make geometry persist |
| `0x009a0204` (CFG1) | **no** | `0x02779000` reverted to `0x02449000` |
| per-FBPA CFG1 | **no** | same sweep |
| per-FBPA CSTATUS | **no** | same |
| `0x00100ce0` (LMR) | **no** | `0x20b` reverted to `0x288` |
| the FB-geometry PLMs | **no** (they re-lock) | |
| `0x001180f0` (AON LMR shadow) | **no** (reverts despite being AON) | |
| `0x008403c4` (SEC2 reset PLM) | **cleared** to `0xff` | the FLR removes the `0x8f` HS taint |

Geometry **does** survive a driver unload and reload with no secondary bus reset: after
unloading, `0x009a0204` still read `0x02669000` and `0x00100ce0` still read `0x0000028a`, and a
fresh load again enumerated 40960 MiB. Full-reset paths (PERST, `nvidia-smi --gpu-reset`,
`echo 1 > /sys/bus/pci/devices/<bdf>/reset`) re-run the signed DevInit with the locked CMP table
and discard everything.

---

## Status codes you may see alongside these registers

| Code | Where | Meaning |
|---|---|---|
| `0xffff` | `kgspExecuteBooterLoad_TU102` return | returned on **every** payload run, success or not. Register readback is the only valid verdict |
| `0x31` | SEC2 `MAILBOX0` | the driver-planted argument left untouched on the raw-exit path that skips `report_status`, **not** a Booter error code |
| `0x15` | Booter status | CSB access error |
| `0x29` | Booter status | `0x001180f8` bits [31:28] non-zero at the `check_1180f8_nibbles (0x1c75)` gate |
| `0x88` | Booter status | `0x001180f8` bits [27:24] non-zero at `check_1180f8_2724 (0x1ba3)` |
| `0x5` | Booter status | `wpr_region_check (0x28ac)` failure, including an empty region |
| `0x47` | Booter status | stack-check failure |
| `0x59` | driver log | `dmem.bin` absent, benign |
| `0x62` | CPU-RM | `NV_ERR_RESET_REQUIRED`; the `RmInitAdapter` triplet `(0x62:0x40:2028)` is the WPR2-already-up case. As a Booter `MAILBOX0` code `0x62` belongs to the PKA paths instead |
| `0x65` | driver | `NV_ERR_TIMEOUT`, the stage-3 handoff poll timing out |
| `0x96` | Booter status | normal |
| `0x24` / `0x25` | CPU-RM | `kbusVerifyBar2` failure / StateLoad reached; the CFG1-versus-LMR discriminator |
| `Xid 31`, `Xid 154`, `Xid 119` | kernel | fatal GPU loss above roughly 40 GB on the `80` branch; multi-context failure at 80 GB; GSP RPC timeout |

See [Troubleshooting](../procedures/troubleshooting.md) for what to do about each.

---

## Related pages

- [How the unlock works](how-it-works.md) and [Overview](overview.md)
- [Privilege level masks](privilege-level-masks.md), [Falcon and Booter](falcon-and-booter.md),
  [The ROP chain](rop-chain.md)
- [Memory geometry](memory-geometry.md), [Compute throttle](compute-throttle.md),
  [PCIe Gen2](pcie-gen2.md), [Driver patches](driver-patches.md)
- [Fuses and OTP](../hardware/fuses-and-otp.md),
  [Memory subsystem](../hardware/memory-subsystem.md),
  [PCIe subsystem](../hardware/pcie-subsystem.md)
- [Glossary](../start/glossary.md) for PLM, FLR, WPR, FBPA, LMR, AON, HS, PL0/PL3
- [Register index](../appendix/register-index.md) for a flat alphabetical address list
