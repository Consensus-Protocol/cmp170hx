# Register index

## What this page covers

This is a **flat lookup table of every BAR0 register address documented anywhere in this
project, sorted by numeric address ascending.** Its only job is to answer one question fast:
you have just seen an unfamiliar hex address in a `dmesg` line, a patch, or somebody's script,
and you want to know what block it belongs to and what it does.

It is deliberately shallow. Every row carries a one-line meaning and a link into the page that
explains the register properly. For stock values, unlocked values, PLM gating, FLR survival and
the reasoning behind any of it, go to [Register reference](../unlock/register-reference.md),
which is the explanatory companion to this index.

Two rules that save the most time:

- **Everything in the Address column is a BAR0 byte offset**, absolute from the start of
  region 0. Several unlock *values* look like plausible addresses and are not. If your number is
  not in this index, check
  [Numbers that are not BAR0 addresses](../unlock/register-reference.md#numbers-that-are-not-bar0-addresses)
  before assuming the index is incomplete.
- **A read that returns `0xbadf....` is not data.** It is a PRI poison or privilege-violation
  sentinel. See
  [Sentinel values](../unlock/register-reference.md#sentinel-values).

Where the project never established what a register does, the row says **not documented**. That
phrase is load-bearing: it means nobody in the archive resolved it, not that it was omitted here.

To read an address by hand on a card at `0000:05:00.0`:

```bash
sudo dd if=/sys/bus/pci/devices/0000:05:00.0/resource0 \
        bs=4 count=1 skip=$((0x009a0204 / 4)) 2>/dev/null | xxd -e -g4
```

### Block abbreviations used below

| Block | Meaning |
|---|---|
| `PMC` / `PBUS` | master control and bus scratch |
| `PTOP` | topology scalars, describing the full GA100 die |
| `XVE` | PCIe config-space shadow, BAR0 base `0x88000` |
| `XP-PL` | PCIe physical-layer link config, `0x0008cxxx` |
| `XP3G` | PCIe PHY-rate override arrays, `0x0008e1xx` |
| `FBHUB` / `MMU` | frame-buffer hub and memory management unit, `0x001xxxxx` |
| `PMU` | power management unit Falcon, `0x0010axxx` |
| `GSP` | GSP RISC-V core and its Falcon shell, `0x0011xxxx` |
| `BSI` | always-on (AON) boot and secure scratch island, `0x001180xx` |
| `LTC` | level-2 cache slices |
| `WPR` | write-protected region control, `0x001fa7xx` / `0x001fa8xx` |
| `SKED` / `FECS` / `SM` | graphics and compute front end |
| `FUSE` | fuse and OTP shadows, `0x0082xxxx` |
| `FEAT_OVR` | feature-override block, `0x008238xx` |
| `SEC2` | security coprocessor Falcon, BAR0 + `0x840000` |
| `FBPA` | frame-buffer partition, broadcast `0x009axxxx` and unicast `0x0090xxxx` |

---

## The index

| Address | Name | Block | One-line meaning | Detail |
|---|---|---|---|---|
| `0x00000000` | `PMC_BOOT_0` | PMC | silicon identity; `0x170000a1` on every valid GA100 | [Topology scalars](../unlock/register-reference.md#topology-scalars-0x0002xxxx) |
| `0x00001404` | `PBUS_SW_SCRATCH(1)` | PBUS | software scratch; `0x20042000`, bit 14 clear on every card surveyed | [Topology scalars](../unlock/register-reference.md#topology-scalars-0x0002xxxx) |
| `0x0002241c` | `NV_PTOP_FS4` | PTOP | bit 0 `GEN2_PCIE`, bit 7 `GEN2_PCIE_SPEED`; `0x00000000` on the 8 GB card, `0x00000081` on the 10 GB card | [PCIe subsystem](../hardware/pcie-subsystem.md) |
| `0x00022430` | `PTOP_SCAL_NUM_GPCS` | PTOP | GPC count of the full die, `8` | [Topology scalars](../unlock/register-reference.md#topology-scalars-0x0002xxxx) |
| `0x00022434` | `PTOP_SCAL_TPC_PER_GPC` (`NUM_TPC_GPC`) | PTOP | TPCs per GPC, `8` | [Topology scalars](../unlock/register-reference.md#topology-scalars-0x0002xxxx) |
| `0x00022438` | `PTOP_SCAL_NUM_FBPS` | PTOP | FBP count of the full die, `0x0000000c` (12) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x0002243c` | `PTOP_SCAL_NUM_FBPAS` | PTOP | FBPA count of the full die, `0x00000018` (24) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x00022454` | `PTOP_SCAL_NUM_LTCS` | PTOP | L2 slice count, `0x00000018` (24) | [Topology scalars](../unlock/register-reference.md#topology-scalars-0x0002xxxx) |
| `0x00022458` | `PTOP_SCAL_FBPA_PER_FBP` | PTOP | FBPAs per FBP, `0x00000002` (an RTX 3090 reads 1) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x0002246c` | `PTOP_SCAL_NUM_NVLINK` | PTOP | NVLink count of the full die, `0x0000000c` (12) | [NVLink hardware](../hardware/nvlink-hardware.md) |
| `0x00022470` | `PTOP_FS_STATUS` | PTOP | floorsweep-status bit vector, `0x0000003f`; bit0 TPC, bit1 GPC, bit2 FBP, bit3 ROP, bit4 FBIO | [Topology scalars](../unlock/register-reference.md#topology-scalars-0x0002xxxx) |
| `0x00085080` | (unnamed) | PRIV | reads `0xbadf1100` from the SEC2 injection point; GSP writes it at a privilege the exploit never reaches | [PROT-walled registers](../unlock/register-reference.md#registers-that-are-prot-walled-or-poisoned-from-the-injection-point) |
| `0x00085084` | (unnamed) | PRIV | same as above | [PROT-walled registers](../unlock/register-reference.md#registers-that-are-prot-walled-or-poisoned-from-the-injection-point) |
| `0x00088070` | (unnamed) | XVE | reads return 0, writes ignored; **not documented** | [PROT-walled registers](../unlock/register-reference.md#registers-that-are-prot-walled-or-poisoned-from-the-injection-point) |
| `0x00088084` | `LINK_CAP` (LnkCap) | XVE | PCIe link capabilities shadow; `0x00456101` stock, `0x00456102` after the Gen2 patch | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x00088088` | `LINK_CTRL_STATUS` (LnkSta) | XVE | negotiated link state; `0x10410040` stock (LnkSta in bits [31:16], LnkCtl in [15:0]), `0x1042xxxx` at Gen2. Speed = `(value >> 16) & 0xF` | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x0008808c` | (unnamed) | XVE | reads 0, writes ignored. **Not** the LnkCap2 mirror, despite one field manual saying so | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x00088090` | (unnamed) | XVE | reads 0, writes ignored; **not documented** | [PROT-walled registers](../unlock/register-reference.md#registers-that-are-prot-walled-or-poisoned-from-the-injection-point) |
| `0x000880a4` | `LINK_CAP2` (LnkCap2) | XVE | supported link speeds vector; `0x00000002` (2.5 GT/s only) stock, `0x00000006` (Gen1+Gen2) after the patch. Hardware read-only to `setpci` | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x000880a8` | `LINK_CTRL_2` (LnkCtl2) | XVE | target link speed; patch sets bits [3:0] = `0x2` and bits [19:16] = `0xF` | [PCIe Gen2](../unlock/pcie-gen2.md) |
| `0x0008841c` | `PRIV_MISC_1` | XVE | Gen2 enable bits; `0x20340500` becomes `0x20342d00` (set 11 and 13, clear 12 and 14). Succeeds first attempt and survives Booter Load | [PCIe Gen2](../unlock/pcie-gen2.md) |
| `0x0008860c` | `VSEC_DEVICE` | XVE | vendor-specific device word; patch wants `0x00000800` to become `0x00000801`, and the **write fails on silicon** | [PCIe Gen2](../unlock/pcie-gen2.md) |
| `0x00088610` | `VSEC_HIERARCHY` | XVE | vendor-specific hierarchy word; `0x00001001` stock, patch clears bit 12 and sets bit 0 by plain host write | [PCIe Gen2](../unlock/pcie-gen2.md) |
| `0x0008872c` | LTSSM override (`XVE_OVR`) | XVE | written `0x00000006` to skip the mid-boot retrain. `0x2` and `0xa` expose extra Gen2 behaviour under VFIO but eventually wedge the function | [PCIe Gen2](../unlock/pcie-gen2.md) |
| `0x00088ab4` | `XVE_B` PLM | XVE | privilege mask, opened to `0xffffffff` by the nine-entry Gen2-family PLM table | [Gen2 PLM table](../unlock/register-reference.md#added-by-the-gen2-family-branches-nine-entries-total) |
| `0x00088ce4` | (unnamed) | XVE | `0x0000003f` on 170HX, `0x00000014` on A100; a VBIOS block computes it by mask-and-merge. Meaning **not documented** | [VBIOS](../hardware/vbios.md) |
| `0x00088fe8` | `XVE_D0` PLM | XVE | privilege mask, opened to `0xffffffff` by `xp3gTable` | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x00088fec` | `XVE_D4` PLM | XVE | privilege mask, opened to `0xffffffff` by `xp3gTable` | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x00088ff0` | `XVE_D8` PLM | XVE | privilege mask, opened to `0xffffffff` by `xp3gTable` | [XVE shadow](../unlock/register-reference.md#xve-config-space-shadow-bar0-base-0x88000) |
| `0x00088ff4` | `XVE` PLM | XVE | privilege mask over the PCIe config shadow; without it host reads return `0xbadf5040` | [Gen2 PLM table](../unlock/register-reference.md#added-by-the-gen2-family-branches-nine-entries-total) |
| `0x00088ff8` | `XVE_C` PLM | XVE | third XVE capability privilege mask, opened to `0xffffffff` | [Gen2 PLM table](../unlock/register-reference.md#added-by-the-gen2-family-branches-nine-entries-total) |
| `0x0008c040` | `LINK_CONFIG_0` | XP-PL | bits [19:18] `MAX_RATE`; patch read-modify-writes them to `0x2` | [XP-PL block](../unlock/register-reference.md#xp-pl-link-config-block-0x0008cxxx) |
| `0x0008c044` / `0x0008c048` / `0x0008c04c` | LINK_CONFIG cluster | XP-PL | a different cluster from the three that work; HS writes to these were rejected. Field layout **not documented** | [XP-PL block](../unlock/register-reference.md#xp-pl-link-config-block-0x0008cxxx) |
| `0x0008c080` | link width register | XP-PL | A100 reads `0x00001010`; never used as a lever on the 170HX. Width is a board-level limit, not this register | [Physical mods](../operations/physical-mods.md) |
| `0x0008c1c0` | `PL_LINK_RATE` | XP-PL | PHY rate word; written `0x00240036` for Gen2 (A100 reads `0x00040036`) | [XP-PL block](../unlock/register-reference.md#xp-pl-link-config-block-0x0008cxxx) |
| `0x0008c2c0` | `CYA_0` | XP-PL | bit 2 is the `DIS_G2` chicken bit and must be cleared. The central Gen2 lever | [PCIe Gen2](../unlock/pcie-gen2.md) |
| `0x0008e100` | `XP3G_STATUS` base | XP3G | four-dword status array, slot *n* at base + 4*n*; read only | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e10c` | `XP3G_STATUS3` | XP3G | slot 3 of the status array | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e110` | `XP3G_OVR0` | XP3G | override-enable slot 0, written `0x00000001` (one-hot per slot) | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e11c` | `XP3G_OVR3` | XP3G | override-enable slot 3, written `0x00000004` | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e120` | `XP3G_VAL0` | XP3G | override value slot 0, written `0x00000000`. Values are always written before enables | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e12c` | `XP3G_VAL3` | XP3G | override value slot 3, written `0x00200000` (the A100 `FUSE_PCIE_MAGIC_D` value) | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e1b0` | `XP3G_PLM` | XP3G | privilege mask over the XP3G block; opens cleanly to `0xffffffff` | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e1b4` | `XP3G_PLM4` | XP3G | second XP3G privilege mask | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e1b8` | `XP3G_PLM8` | XP3G | third XP3G privilege mask | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x0008e1bc` | `XP3G_PLMC` | XP3G | fourth XP3G privilege mask | [XP3G block](../unlock/register-reference.md#xp3g-phy-rate-override-block-0x0008e1xx-0x0008e1xx) |
| `0x00100800` | `FBHUB_NUM_ACTIVE_LTCS` | FBHUB | active L2 slice count, `0x10` (16) on the 8 GB card / `0x14` (20) on the 10 GB card and on A100 PCIe 40/80 GB | [MMU / FB hub](../unlock/register-reference.md#mmu-fb-hub) |
| `0x00100b10` | FB-geometry PLM | FBHUB | one of the five `FB_GEO_PLMS` the clean-room chain opens; `0xffffff8f` locked. Opening it does **not** make geometry survive FLR | [FB-geometry PLM set](../unlock/register-reference.md#the-fb-geometry-plm-set-clean-room-tools-only) |
| `0x00100b38` | FB-geometry PLM | FBHUB | sixth entry in the earliest HS recipe only | [FB-geometry PLM set](../unlock/register-reference.md#the-fb-geometry-plm-set-clean-room-tools-only) |
| `0x00100b84` | PLM candidate | FBHUB | reads `0xffffff88`; what it guards is **not documented** | [26-register PLM survey](../unlock/register-reference.md#the-26-register-plm-survey) |
| `0x00100b90` | `FBHUB_MEM_PART_BCFG0` | FBHUB | memory-partition broadcast config, `0x00000603` on every card | [MMU / FB hub](../unlock/register-reference.md#mmu-fb-hub) |
| `0x00100b98` | `SYSMEM_HSHUB_CONNECTION_CFG` | FBHUB | sysmem routing, `0x00000003` (BOTH, PCIe) | [MMU / FB hub](../unlock/register-reference.md#mmu-fb-hub) |
| `0x00100b9c` | PLM candidate | FBHUB | reads `0xffffffcf`; what it guards is **not documented** | [26-register PLM survey](../unlock/register-reference.md#the-26-register-plm-survey) |
| `0x00100ce0` | MMU local memory range (LMR) | MMU | **total FB size seen by the MMU.** One of the two geometry writes. `0x00000208` / `0x00000288` stock, `0x0000020B` (64 GB) / `0x0000028A` (40 GB) unlocked. Encoding `MiB = MAG[9:4] << SCALE[3:0]` | [Memory geometry](../unlock/memory-geometry.md) |
| `0x00100ec0` | `MMU_NUM_ACTIVE_LTCS` | MMU | `0x05001414` on the 10 GB SKU and all three A100 SKUs; `0x04001410` reported on the 8 GB SKU. The per-SKU split is an **open question**, not a dissent: `...1410` is consistent with 16 LTCs, `...1414` with 20 | [MMU / FB hub](../unlock/register-reference.md#mmu-fb-hub) |
| `0x0010a040` | PMU `FALCON_MAILBOX0` | PMU | PMU Falcon mailbox 0; PL0-writable, read `0x00000300` | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x0010a044` | PMU `FALCON_MAILBOX1` | PMU | PMU Falcon mailbox 1; PL0-writable, read `0x00000000` | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00110040` | GSP `FALCON_MAILBOX0` | GSP | plain 32-bit register, not a FIFO. PL0-writable, reset to 0 by a healthy GSP boot. **This is not the mailbox `s_executeBooter` reads** (that is SEC2's at `0x00840040`) | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00110044` | GSP `FALCON_MAILBOX1` | GSP | PL0-writable, reads `0x00000000` | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00110180` / `0x00110184` | GSP `IMEMC` / `IMEMD` | GSP | GSP instruction-memory port pair | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x001101c0` / `0x001101c4` | GSP `DMEMC` / `DMEMD` | GSP | GSP data-memory port pair, carries WPR addresses | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00110624` | GSP `FBIF_CTL` | GSP | aperture control; the Booter's `reg_init` writes `0x90` (`ALLOW_PHYS_NO_CTX` bit 7 plus bit 4) | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x00110684` | GSP FBIF companion | GSP | written `1` by `reg_init`; purpose **not documented** | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x00111040` | GSP Falcon-shell `MAILBOX0` | GSP | distinct from `0x00110040`; PL0-writable, reads `0x00000000` | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00111240` | `RISCV_STATUS` | GSP | GSP core status. Non-zero means the RISC-V core started (`0x35` and `0x33` both reported on healthy boots); `0x0` means it never started | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x00111268` | `RISCV_CPUCTL` | GSP | GSP core control | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x0011126c` | GSP RISC-V companion | GSP | written `1` by `reg_init`; purpose **not documented** | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x001180f0` | AON LMR shadow | BSI | always-on shadow of the memory-range value. **Reverts on FLR**, so it is not a persistence lever | [FLR survival](../unlock/register-reference.md#the-flr-survival-table) |
| `0x001180f8` | `NV_PGC6_BSI_SECURE_SCRATCH_14` | BSI | bit 26 = `BOOT_STAGE_3_HANDOFF`. Set GPU-side by SEC2 in HS context; the host driver only polls it, and boot hang `0x65` is that poll timing out. **The shipping chain never writes it** | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x00118244` / `0x00118248` | WPR staging pair | BSI | read then zeroed by `booter_load_wpr_main` | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x0011824c` / `0x00118250` | memcfg handoff | BSI | written by `memcfg_program`; the apply poll runs only if `0x0011824c` bit 0 is set | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x001182d0` | AON secure scratch | BSI | reachable at PL3; contents **not documented** | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x00118f78` | auxiliary scratch | BSI | reads `0x00000000` on every card surveyed; purpose **not documented** | [GSP and BSI](../unlock/register-reference.md#gsp-risc-v-and-bsi-secure-scratch-bar0-0x110000-0x118000) |
| `0x00120078` | `RING_ENUM_GPC` | PRIV ring | reads `5` on every 170HX; never moved by any write attempt | [GA100 silicon](../hardware/ga100-silicon.md) |
| `0x001402b4` | LTC companion | LTC | a write of `0x00a00030` was attempted and did not move the 40 GiB fold. Field layout **not documented** | [The 80 GB question](../frontier/80gb.md) |
| `0x0017e22c` | L2/LTC address-map register | LTC | `0x00280404` native; never programmed by anything, yet 40 GB works | [L2 / LTC](../unlock/register-reference.md#l2-ltc) |
| `0x0017e2a0` / `0x0017e2a4` | per-LTC decode | LTC | targeted by the clean-room v8 tool; on the 170HX `DECODE_VAL` stays `0x70000300` throughout, which remains unexplained | [The 80 GB question](../frontier/80gb.md) |
| `0x001fa7c4` | `WPR_PLM` | WPR | privilege mask over the WPR region registers. `0x0004cb8f` locked; **shipping PLM index 2, opened to `0xffffffff`** | [PLM table](../unlock/register-reference.md#written-by-shipping-master-four-entries-in-this-order-up-to-two-attempts-each) |
| `0x001fa7c8` | `MMU_LOCK` PLM | WPR | write nibble `0x8` means L3/HS only; `0x0004cb8f`. Read only in this project | [WPR block](../unlock/register-reference.md#wpr-block-0x001fa7xx-0x001fa8xx) |
| `0x001fa7cc` | `WPR_CFG_PLM` | WPR | privilege mask over the WPR allow-masks. **Shipping PLM index 0, opened to `0xfffff0ff`, not `0xffffffff`.** This exception is real, believe the patch | [PLM table](../unlock/register-reference.md#written-by-shipping-master-four-entries-in-this-order-up-to-two-attempts-each) |
| `0x001fa814` | WPR read-allow mask | WPR | mode field in bits [7:4]; the Booter sets bit `0x800` under mask `0x0ffff8ff` | [WPR block](../unlock/register-reference.md#wpr-block-0x001fa7xx-0x001fa8xx) |
| `0x001fa818` | WPR write-allow mask | WPR | as above | [WPR block](../unlock/register-reference.md#wpr-block-0x001fa7xx-0x001fa8xx) |
| `0x001fa81c` / `0x001fa820` | `WPR1_ADDR_LO` / `HI` | WPR | WPR1 range, value in bits [31:4] shifted left 12; cleared by the clean-room re-fire chain | [WPR block](../unlock/register-reference.md#wpr-block-0x001fa7xx-0x001fa8xx) |
| `0x001fa824` | `WPR2_ADDR_LO` | WPR | **saved before the PLM loop and re-armed before every Booter attempt**, because a second `booter_load` otherwise aborts with "WPR2 already up" (status `0x62`). Empty/INIT reads `0x0fffffff` | [How the unlock works](../unlock/how-it-works.md) |
| `0x001fa828` | `WPR2_ADDR_HI` | WPR | same pairing; `HI = 0` makes `kgspIsWpr2Up()` return false. Empty/INIT reads `0` | [How the unlock works](../unlock/how-it-works.md) |
| `0x001fa82c` / `0x001fa830` | memlock range LO / HI | WPR | `0x1ffffff0` / `0x00000000` post-AHESASC (empty); read only | [WPR block](../unlock/register-reference.md#wpr-block-0x001fa7xx-0x001fa8xx) |
| `0x00407000` | `SKED_HW_BLK` | SKED | `0x00004042` with a driver, `0xbadf1201` without | [Graphics, SKED and FECS](../unlock/register-reference.md#graphics-sked-and-fecs-investigated-never-used) |
| `0x00407010` | `SKED_PM_UNK10` | SKED | reads `0x00000000`; meaning **not documented** | [Graphics, SKED and FECS](../unlock/register-reference.md#graphics-sked-and-fecs-investigated-never-used) |
| `0x00407020` | `SKED_TRAP` | SKED | reads `0x00000000` | [Graphics, SKED and FECS](../unlock/register-reference.md#graphics-sked-and-fecs-investigated-never-used) |
| `0x00407024` | `SKED_TRAP_EN` | SKED | `0x3dfffffc`, identical to A100 | [Graphics, SKED and FECS](../unlock/register-reference.md#graphics-sked-and-fecs-investigated-never-used) |
| `0x00407054` | `SKED_UNK54` | SKED | `0x60000600` pre-driver or `0x600000c0`, and **zero on both A100 and RTX 3090**. The most-referenced undocumented SKED register in the GSP firmware. Never write-tested; function **not documented** | [Open questions](../frontier/open-questions.md) |
| `0x00408970` | `gpcMask` | GR | `0xdc` on one card and re-asserts after every forcing attempt. A closed dead end | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00409664` | `FECS_FEAT_OVERRIDE` | FECS | returns `0xbadf5040` on **every** Ampere card ever probed, throttled or not, so the value carries no information about this card | [Compute throttle](../unlock/compute-throttle.md) |
| `0x00409668` | `FECS_FEAT_READOUT_1` | FECS | same, `0xbadf5040` everywhere | [Compute throttle](../unlock/compute-throttle.md) |
| `0x00504204` | `SM_ISSUE_RATE_MODIFIER` | SM | **not** the compute throttle: reads `0x00000005` on 13 comparison Ampere cards and on a 96-SM GA100 with every speed-select fuse at 0. Host-writable; zeroing it changes nothing. `0xbadf1201` with no driver loaded | [Compute throttle](../unlock/compute-throttle.md) |
| `0x00820000` | `FUSE_FUSECTRL` | FUSE | fuse controller, `0xe0040000` identical on all 15 cards in the cohort | [Fuses and OTP](../hardware/fuses-and-otp.md) |
| `0x00820040` | `FUSE_EN_SW_OVERRIDE` | FUSE | `0x00000000` on 170HX and A100, `0x00000001` on consumer and engineering-sample parts. Writable and persistent on the 170HX but changes nothing observable, which is what rules out the software fuse-override route | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x00820078` | `FUSE_EN_PROGRAM` | FUSE | `0x00000001` on all 15 cards | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x0082007c` | `FUSE_DIS_PROGRAM` | FUSE | `0x00000000`; `0xbadf5040` on GA10x | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x00820080` | `FUSE_BYPASS_STATUS` | FUSE | `0x00000000`; `0xbadf5040` on GA10x | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x00820084` | `FUSE_DIS_SW_OVR` | FUSE | `0x00000001` on all 15 cards; HS writes bounce. Software fuse override is permanently blocked | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x008200d0` .. `0x008200f4` | `OPTB_D0` .. `OPTB_F4` PLMs | FUSE | ten consecutive privilege masks (`d0`, `d4`, `d8`, `dc`, `e0`, `e4`, `e8`, `ec`, `f0`, `f4`), all written `0xffffffff` by the Gen2 `xp3gTable`. `0x008200d0` and `0x008200dc` read `0xffffff8f` locked, the other eight read open. **What each one guards is not documented** | [OPTB PLM block](../unlock/register-reference.md#optb-plm-block-written-by-0007) |
| `0x008200fc` | `FUSE_SS_PLM` / `OPT_PLM` | FUSE | one register, two names (`OPT_PLM` is the branch-code label, `FUSE_SS_PLM` the clean-room tooling name). Guards the speed-select fuse block and `OPT_FB_CONFIG`. **Never written by shipping master.** Read `0xffffffff` in one sweep and `0x000003ff` in another, and whether it is writable is **still open** | [Gen2 PLM table](../unlock/register-reference.md#added-by-the-gen2-family-branches-nine-entries-total) |
| `0x00820148` | OTP spare bit | FUSE | `0x00000000`, never settable; purpose **not documented** | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x00820224` | `FUSE_SS_DP` | FUSE | double-precision speed-select fuse, a separate 1-bit field: `0x00000001` (reduced) on the 170HX | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008202c4` | `OPT_ROP_L2_DISABLE` | FUSE | mirrors `0x00820368` | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820328` | `OPT_FB_CONFIG` | FUSE | 4-bit memory topology selector, guarded by PLM `0x008200fc`. Documented in `probe.sh`, never write-tested | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x00820340` | `OPT_MEMORY_LOCKED_ENABLED` (`FUSE_MEM_LOCKED`) | FUSE | `0x00000001` on all 15 cards in the cohort, meaning memory config is nominally not runtime-changeable. It does not block the unlock: the shipping chain rewrites CFG1 and the LMR anyway | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x00820350` | `OPT_GPC_DISABLE` | FUSE | per-card GPC disable mask: `0x85`, `0x45`, `0x13`, `0xa8` on four different cards. HS writes bounce, the value is latched | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820364` | `OPT_FBP_DISABLE` | FUSE | FBP disable mask: `0x00000840` on the 10 GB card (FBP 6 and 11 off), `0x00000852` on a community dump, `0x00000009` and `0x00000180` on two other units | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820368` | `OPT_FBPA_DISABLE` | FUSE | FBPA disable mask: `0x000000c3` on the 10 GB card (20 live), `0x00c0330c` on the 8 GB card (16 live). **This, not CFG1, is what fixes the FBPA count** | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x0082036c` | `OPT_FBIO_DISABLE` | FUSE | mirrors `0x00820368` | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x0082038c` | `FUSE_QUADRO_WR_SEC` | FUSE | `0x00000001`; this is what permits `0x00823804` to be opened at all | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x00820394` | `OPT_PCIE_LANE_DISABLE` | FUSE | `0x00000000` on the 170HX and every comparison part. **Proof that the x4 width is a board-level capacitor issue, not a fuse** | [PCIe subsystem](../hardware/pcie-subsystem.md) |
| `0x00820398` | `OPT_SPARE_FS` | FUSE | `0x00000000` | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x008203a4` | `OPT_SYS_PIPE_DISABLE` (inferred) | FUSE | inferred syspipe / SMC-engine disable mask: `0xfe` on both 170HX units (if the inferred mapping is correct, bits 1–7 disable syspipes 1–7), `0x0` on GA106 (RTX 3060); no A100 reading yet. In the floorsweep-fuse readback block. Host BAR0 writes do not latch (write-probe `0xfc` read back `0xfe`). **Matches the single `1g.64gb` MIG profile** | [Fuses and OTP](../hardware/fuses-and-otp.md) |
| `0x008203f0` | `FUSE_FEAT_OVR_DIS` (`OPT_FEATURE_FUSES_OVERRIDE_DISABLE`) | FUSE | **the master kill fuse, unblown at `0x00000000`. This single zero is why the entire unlock exists** | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x008203f4` | `OPT_INTERNAL_SKU` | FUSE | `0` | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x0082049c` | `OPT_HALF_FBPA_ENABLE` | FUSE | 24-bit per-FBPA half-capacity bitmask; non-zero means capacity halved. From the `probe.sh` catalogue | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x008204bc` | `OPT_SLT_REV` | FUSE | silicon lot/test revision, read by `ga100_topology_report.py` | [Fuses and OTP](../hardware/fuses-and-otp.md) |
| `0x008204d8` | `OPT_PCIE_DEVIDA` | FUSE | SKU identity fuse: `0x000020c2` (8 GB), `0x00002082` (10 GB); A100 reads `0x20b2` | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x00820520` | `FUSE_PCIE_MAGIC_D` | FUSE | `0x16680000` on the 170HX with bit 25 set (`GEN4_SPEED_DISABLED`, NVIDIA bug 2220334) versus `0x00200000` on A100 and Drive GA100. **Whether it is writable is an open problem** | [Gen3 and Gen4](../frontier/pcie-gen3-gen4.md) |
| `0x0082056c` | `OPT_PCIE_DEVIDB` | FUSE | `0x000020c2` on both physical 10 GB units, so on the 10 GB card DEVIDA and DEVIDB disagree. The 8 GB value is **disputed**: one 2026-07-19 probe of a `0x20c2` card reads `0x000020c2`, while the `DEVIDB = DEVIDA + 0x40` rule that holds on all 11 parts with data predicts `0x00002102` | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x0082057c` | `FUSE_PCIE_GEN23_DIS` (`OPT_PCIE_BOOT_GEN23_DISABLE`) | FUSE | `0x00000001` on both 170HX SKUs, `0x00000000` on 14 other Ampere parts. **Hard read-only**: attempted from host, HS ROP and the Booter payload, always reads back `0x00000001`. Gen2 works anyway | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x00820580` | `FUSE_PCIE_GEN3_DIS` (`OPT_PCIE_BOOT_GEN3_DISABLE`) | FUSE | `0x00000001` on both 170HX SKUs | [Gen3 and Gen4](../frontier/pcie-gen3-gen4.md) |
| `0x00820584` | `FUSE_DEVID_SW_OVR_DIS` | FUSE | `0x00000001` on the 170HX and every comparison part | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x0082059c` | `FUSE_SS_FFMA` | FUSE | fused-multiply-add speed select, `0x00000005` (divide by 32) on the 170HX | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008205c4` | `OPT_GPC_DEFECTIVE` | FUSE | `0x00000000` on several cards whose DISABLE mask had three bits set, `0x81` on one 10 GB card. "Disabled" and "defective" are separate masks | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x008205cc` | `OPT_FBP_DEFECTIVE` | FUSE | `0x00000840` on the 10 GB card, exactly matching `OPT_FBP_DISABLE`, so no FBP is disabled-but-good on that unit | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x008205d0` / `0x008205d4` / `0x008205e8` | `OPT_FBPA_DEFECTIVE` / `FBIO_DEFECTIVE` / `ROP_L2_DEFECTIVE` | FUSE | `0x00c03000` each | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820618` | `FUSE_FBPA_MEM_WR_SEC` (`OPT_SECURE_FBPA_MEM_WR_SECURE`) | FUSE | `0x00000001` on all 15 cards | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x00820670` | `OPT_FB_FALCON_PRI_ACCESS_DISABLE` | FUSE | `0x00000000`, meaning **a Falcon retains PRI access to FB registers**. That property is exactly what the SEC2 Booter ROP chain depends on | [The ROP chain](../unlock/rop-chain.md) |
| `0x00820684` | `FUSE_NVLINK_DIS` (`OPT_NVLINK_DISABLE`) | FUSE | `0x00000007`, all three bits of [2:0] set, versus `0x00000000` on A100 and most consumer parts | [NVLink](../frontier/nvlink.md) |
| `0x0082074c` | `FUSE_OPT_SECURE_GSP` | FUSE | `0x00000001` on all 15 cards | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x008207d4` | `FUSE_SS_FMLA16` | FUSE | `0x00000005` on 170HX, `0x00000000` on every unthrottled Ampere part | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008207d8` | `FUSE_SS_FMLA32` | FUSE | `0x00000005`; an RTX 3070 reads 1 | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008207dc` | `FUSE_SS_IMLA0` | FUSE | `0x00000005` | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008207e0` | `FUSE_SS_IMLA1` | FUSE | `0x00000005` | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008207e4` | `FUSE_SS_IMLA2` | FUSE | `0x00000005` | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008207e8` | `FUSE_SS_IMLA3` | FUSE | `0x00000005` | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x008207ec` | `FUSE_SS_IMLA4` | FUSE | `0x00000005`; an RTX 3070 reads 1. All nine speed-select fuses stay `0x5` after the unlock, because the override supersedes them | [SM speed-select fuses](../unlock/register-reference.md#sm-speed-select-fuses-the-throttle-itself) |
| `0x00820800` | `CTRL_OPT_HALF_FBPA` | FUSE | merged override state for the half-FBPA fuse, from the `probe.sh` catalogue | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x00820818` | `CTRL_OPT_FBPA` | FUSE | `0x00000000`, no override present | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820820` | `CTRL_OPT_PERLINK` | FUSE | per-NVLink override shadow; never write-tested | [NVLink fuses](../unlock/register-reference.md#nvlink-fuses) |
| `0x0082082c` | `CTRL_OPT_PCIE_LANE` | FUSE | `0x00000000` | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x00820834` | `CTRL_OPT_FB_CONFIG` | FUSE | merged override state for `OPT_FB_CONFIG` | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x00820838 + i*4` | `FUSE_CTRL_OPT_TPC_GPC(i)` | FUSE | per-GPC TPC override, `0x00000000`. **Remove-only (subtractive)**: writing it never adds a TPC back | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820840` | MIG enable | FUSE | `0` stock; setting bit 0 was reported to enable MIG and persist. **Single report, and a repo-wide grep for `0x820840` returns nothing** | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820938` | `CTRL_OPT_FBP` | FUSE | `0x00000000` | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x008209b8` | `CTRL_OPT_NVLINK` | FUSE | bits [15:0], per link; reads `0x00000000` on every card probed | [NVLink fuses](../unlock/register-reference.md#nvlink-fuses) |
| `0x00820c00` | `STATUS_HALF_FBPA` | FUSE | `0`, so there are no half-capacity fuses to recover | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820c14` | `STATUS_OPT_FBIO` | FUSE | `0x00c0330c` on the 8 GB card. **This is FBIO, not FBPA** | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820c18` | `STATUS_OPT_FBPA` | FUSE | `0x00c0330c` / `0x000000c3`. This is the correct address for the FBPA status mirror | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820c1c` | `STATUS_OPT_GPC` | FUSE | always mirrors `0x00820350` | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820c2c` | `STATUS_OPT_PCIE_LANE` | FUSE | `0x00000000` | [PCIe fuses](../unlock/register-reference.md#pcie-fuses) |
| `0x00820c30` | `STATUS_OPT_SPARE_FS` | FUSE | read-only mirror of `OPT_SPARE_FS` | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820c34` | `STATUS_OPT_FB_CONFIG` | FUSE | read-only mirror of `OPT_FB_CONFIG` | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x00820c38 + i*4` | `FUSE_STATUS_OPT_TPC_GPC(i)` | FUSE | per-GPC TPC status; GPC0/3/5 read `0xff` and the others `0x01` on one card | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820d38` | `STATUS_FBP` | FUSE | `0x00000180` on one unit | [Floorsweep fuses](../unlock/register-reference.md#floorsweep-fuses) |
| `0x00820db8` | `STATUS_OPT_NVLINK` | FUSE | `0x00000007`, read-only mirror of `0x00820684`; shared with the Drive A100 | [NVLink fuses](../unlock/register-reference.md#nvlink-fuses) |
| `0x00820e30` | syspipe mask, possible applied copy (inferred) | FUSE | same value as `0x008203a4`: `0xfe` on both 170HX, not decoded (`0xbadf5040`) on GA106; on the 170HX `0x00820e24..e2c` and `0x00820e3c` read `0xbadf5040`, `0x00820e34` reads `0x1`. Host BAR0 write-probe `0xfc` did not latch — not a writable shadow | [Fuses and OTP](../hardware/fuses-and-otp.md) |
| `0x00821060` | `OPT_SKU_ID` | FUSE | `0x00000080` on the 8 GB card (`0x20C2`); `0x00000068` on the 10 GB card (`0x2082`) | [Fuse control](../unlock/register-reference.md#fuse-control) |
| `0x00823800` | `FEAT_OVR_ECC_PLM` | FEAT_OVR | privilege mask, `0xffffff8f` stock. **A distinct register from `0x00823804`**, and a frequent transcription slip. Opened by the Gen2 `xp3gTable`, never by master | [Feature override](../unlock/register-reference.md#feature-override-and-compute-0x008238xx) |
| `0x00823804` | `FEAT_OVR_PLM` | FEAT_OVR | **the privilege mask that gates SS0/SS1.** `0xffffff8f` stock, shipping PLM index 3, opened to `0xffffffff`. The only entry in the always-on island, so it **survives FLR** | [Compute throttle](../unlock/compute-throttle.md) |
| `0x00823808` | `FEAT_OVR_QUADRO` | FEAT_OVR | **per-die and unexplained.** Observed: `0x00100183` (stock, PLM range scan, medium), `0x00000081` (post-unlock probe, medium), `0x00000181` / `0x00000182` (two physical 170HX units, high, one of the 13 binning differences), `0x01000282` (A100 80 GB). Read only. **Open question:** why the value differs across all three dumps. Something in the unlock or the driver may be touching the Quadro-versus-consumer classification word, which could be the lever for driver-visible feature classes. Next step: re-read this register before and after each stage of the shipping sequence on one card | [Feature override](../unlock/register-reference.md#feature-override-and-compute-0x008238xx) |
| `0x0082380c` | `FEAT_OVR_ECC` | FEAT_OVR | `0x00888888`; read only | [ECC](../frontier/ecc.md) |
| `0x00823810` | `FEAT_OVR_ECC_1` | FEAT_OVR | `0x002aaaaa`; read only | [ECC](../frontier/ecc.md) |
| `0x00823814` | `FEAT_READOUT_0` | FEAT_OVR | `0x00000233` read-only on the 170HX; a reference GA100 board read `0xef8ff100`. **Field layout not documented** | [Feature override](../unlock/register-reference.md#feature-override-and-compute-0x008238xx) |
| `0x00823818` | `FEAT_READOUT_1` | FEAT_OVR | `0x016db6ed` throttled, **`0x00000000` unlocked. The cleanest single-register "is this card unlocked" test**, and far more reliable than reading SS0 back | [Verify](../procedures/verify.md) |
| `0x0082381c` | `FEAT_OVR_SM_SPEED_SELECT` (SS0) | FEAT_OVR | **compute unlock write 0.** Eight 4-bit fields (IMLA0-3, FMLA16, FMLA32, FFMA, DP), written `0x88888888` = override enabled, full rate. Stock values vary per card. **Survives FLR** | [Compute throttle](../unlock/compute-throttle.md) |
| `0x00823820` | `FEAT_OVR_SM_SPEED_SELECT_1` (SS1) | FEAT_OVR | **compute unlock write 1.** The ninth field, IMLA4, written `0x00000008`. Both writes are required. **Survives FLR** | [Compute throttle](../unlock/compute-throttle.md) |
| `0x00823824` | `FEAT_OVR_ROW_REMAP` | FEAT_OVR | `0x00000000` on both 170HX SKUs; read only | [Feature override](../unlock/register-reference.md#feature-override-and-compute-0x008238xx) |
| `0x00823828` | `FEAT_READOUT_2` | FEAT_OVR | `0x00000000` on the 170HX, `0x00000007` on all A100 and Drive parts; read only | [Feature override](../unlock/register-reference.md#feature-override-and-compute-0x008238xx) |
| `0x0082382c` | `FEAT_READOUT_2` (alias in one dump) | FEAT_OVR | `0x0000000a`. **Naming unsettled between two dumps**, and the field layout is not documented | [Feature override](../unlock/register-reference.md#feature-override-and-compute-0x008238xx) |
| `0x00823b00` | row-remapper PLM (`FEAT2`) | FEAT_OVR | `0xffffff8f` stock, opened to `0xffffffff` by the Gen2-family patch only. One in-HS sweep read it open after FLR, so it may be AON, but opening it did **not** make geometry persist | [Gen2 PLM table](../unlock/register-reference.md#added-by-the-gen2-family-branches-nine-entries-total) |
| `0x00830040` | NVDEC `MAILBOX0` | NVDEC | blocked / read-only from PL0, reads `0xbadf1100` | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00840040` | SEC2 `FALCON_MAILBOX0` | SEC2 | **the mailbox `s_executeBooter` actually reads.** `0x31` on every payload-carrying run, which is the driver-planted argument left untouched on the raw-exit path, not a Booter error code. Register readback is the only valid verdict | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00840044` | SEC2 `FALCON_MAILBOX1` | SEC2 | second mailbox; blocked / read-only from PL0 | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x0084007c` | `SFTRESET` | SEC2 | soft reset: write 1 and read back, valid only if `SCTL` HSMODE (bit 1) is set | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840084` | `FALCON_RM` | SEC2 | resource-manager scratch | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x008400f4` | `FALCON_HWCFG2` | SEC2 | bit 10 = RISCV, reads **0**, confirming SEC2 is a Falcon v4 core and not RISC-V | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00840100` | `FALCON_CPUCTL` | SEC2 | bit 1 = STARTCPU pulse, bit 4 = HALTED (read-only) | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840104` | `FALCON_BOOTVEC` | SEC2 | boot vector | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x0084010c` | `FALCON_DMACTL` | SEC2 | poll until scrub bits `0x6` clear; `0xffffffff` reads mean the window is not responding yet, not failure | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840110` | `FALCON_DMATRFBASE` | SEC2 | DMA base | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840114` | `FALCON_DMATRFMOFFS` | SEC2 | DMEM/IMEM offset | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840118` | `FALCON_DMATRFCMD` | SEC2 | DMA command | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x0084011c` | `FALCON_DMATRFFBOFFS` | SEC2 | frame-buffer offset | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840128` | `FALCON_DMATRFBASE1` | SEC2 | DMA base high | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840180` / `0x00840184` | `IMEMC` / `IMEMD` | SEC2 | IMEM port pair, auto-incrementing, per-256-byte tags; secure tag bit is `1 << 28`. Used to read the loaded Booter back out over range 0 to `0x8700` | [The ROP chain](../unlock/rop-chain.md) |
| `0x00840240` | `SCTL` | SEC2 | secure control; HSMODE = bit 1, `AUTH_EN` = `1 << 14`. Observed `0x3000` to `0x3002` after an engine reset | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x00840284` | SEC2 `DMEM_PLM` | SEC2 | DMEM privilege mask; `0xff` (fully open) in LS mode | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x008403c0` | `FALCON_ENGINE` | SEC2 | bit 0 = RESET; pulse 1 then 0. The engine-reset gate is `(resetPLM & 0x77) == 0x77` | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x008403c4` | SEC2 reset PLM | SEC2 | **decides whether SEC2 can be reset again, and every clean-room fire tool reads it as a readiness gate.** `0xff` clean, `0xdf` after a successful `booter_unload`, `0x8f` after `secure_teardown` (which blocks `SFTRESET`). `reset_allowed = {0xff, 0xdf}`. **Cleared to `0xff` by FLR** | [Recovery](../procedures/recovery.md) |
| `0x00840480` / `0x00840484` | SEC2 post-fire state | SEC2 | move `0` to `0x1` and `0` to `0x11100` as an HS-exit side effect, never restored. Field layout **not documented** | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00840530` | `SCP_P2PRX` | SEC2 | poll bit 3 during a driverless reset | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x008411ec` | `KFUSE_CTL` | SEC2 | poll bit 0 set and bit 1 clear | [SEC2 Falcon](../unlock/register-reference.md#sec2-falcon-bar0-0x840000) |
| `0x00900200 + n*0x4000` | per-FBPA `CFG0` | FBPA | unicast instance *n* (n = 0..23) of the CFG0 register; `0x07981800` on every live instance on both SKUs | [Per-FBPA aperture](../unlock/register-reference.md#per-fbpa-unicast-aperture) |
| `0x00900204 + n*0x4000` | per-FBPA `CFG1` | FBPA | unicast addressing-depth register. **The shipping driver never writes it**; a single broadcast write to `0x009a0204` suffices in the driver path. In a driverless runtime with no devinit, all 24 must be written by hand | [Per-FBPA aperture](../unlock/register-reference.md#per-fbpa-unicast-aperture) |
| `0x0090020c + n*0x4000` | per-FBPA `CSTATUS_RAMAMOUNT` | FBPA | **the verification target**: `0x200` stock (512 MiB per FBPA), `0x800` at the 40 GB tier, `0x1000` at the 64 GB tier. A floorswept FBPA returns a `0xbadf20xx` sentinel | [Verify](../procedures/verify.md) |
| `0x009a0008` | FB-geometry PLM | FBPA | `0xffffff8f` locked; in the clean-room `FB_GEO_PLMS` list. What exactly it guards is **not documented** | [FB-geometry PLM set](../unlock/register-reference.md#the-fb-geometry-plm-set-clean-room-tools-only) |
| `0x009a000c` | FB-geometry PLM | FBPA | as above | [FB-geometry PLM set](../unlock/register-reference.md#the-fb-geometry-plm-set-clean-room-tools-only) |
| `0x009a0040` | FBFLCN `MAILBOX0` | FBPA | FB Falcon mailbox; blocked / read-only from PL0, reads `0x00003fff` | [Falcon and Booter](../unlock/falcon-and-booter.md) |
| `0x009a0148` | **FBPA PLM** | FBPA | the privilege mask that gates CFG1. `0xffffff8f` stock, **shipping PLM index 1, opened to `0xffffffff`**. Also the built-in fallback payload target when `dmem.bin` is absent | [PLM table](../unlock/register-reference.md#written-by-shipping-master-four-entries-in-this-order-up-to-two-attempts-each) |
| `0x009a014c` | FB-geometry PLM | FBPA | `0xffffff8f`; clean-room list only | [FB-geometry PLM set](../unlock/register-reference.md#the-fb-geometry-plm-set-clean-room-tools-only) |
| `0x009a0164` | `FBPA_NUM_ACTIVE` (`NUM_ACTIVE_FBPS`) | FBPA | `0x00000008` on the 8 GB card | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0168` | PLM candidate | FBPA | reads `0xffffffcf`; appears in the 26-register survey only, and what it guards is **not documented** | [26-register PLM survey](../unlock/register-reference.md#the-26-register-plm-survey) |
| `0x009a0200` | `FBPA_CFG0_BROADCAST` | FBPA | `0x07981800` on 170HX and A100 40/80 GB; `0x06981800` on a reference GA100 Drive part | [FBPA aperture](../unlock/register-reference.md#fbpa-broadcast-aperture-0x009a0000-to-0x009a3fff) |
| `0x009a0204` | `NV_PFB_FBPA_CFG1` (broadcast) | FBPA | **addressing depth per memory partition, and the single most-cited register in the archive.** `0x02449000` stock on both SKUs, `0x02779000` (64 GB) / `0x02669000` (40 GB) unlocked. Tier byte at bits [23:16]: `0x44` / `0x66` / `0x77`. **Does not survive FLR** | [Memory geometry](../unlock/memory-geometry.md) |
| `0x009a020c` | `FBPA_CSTATUS` broadcast | FBPA | `0x00001000` on an unlocked 170HX versus `0x00000fff` on A100 80 GB | [FBPA aperture](../unlock/register-reference.md#fbpa-broadcast-aperture-0x009a0000-to-0x009a3fff) |
| `0x009a0224` | `TIMING1` | FBPA | programmed HBM timing, `0x12050d12` (R2W 18, W2R 13, R2P 5, W2P 18) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0290` | `CONFIG0` | FBPA | `0x1255b93c`; bit 31 `USE_TIMING_REGS` = 0, so the generated shadows are in force | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0294` | `CONFIG1` | FBPA | `0x38d4841b` (CL 27, WL 8, RD_RCD 18, WR_RCD 13, QPOP_OFFSET 14) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0298` | `CONFIG2` | FBPA | `0x88130b11` (tWR 19, W2R_BUS 8, R2W_BUS 8, RPRE 1, WPRE 1, CDLR 11) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a02b0` | `TIMING0_GEN` | FBPA | generated shadow actually in force: tRC 60, tRFC 441, tRAS 42 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a02b4` | `TIMING1_GEN` | FBPA | R2W 29, W2R 20, W2P 28 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a02b8` | `TIMING2_GEN` | FBPA | RD_RCD 18, WR_RCD 13, RRD 6 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a02c0` | `TIMING4_GEN` | FBPA | FAW 21; raw `TIMING4` holds a stale FAW of 40 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a02d8` | `TIMING9_GEN` | FBPA | CCDL 4, CCDS 2 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a02e0` | `TIMING16_GEN` | FBPA | RP 18 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0300` | `FBPA_MRS_0` | FBPA | HBM mode register 0, `0x00000003` on 170HX, A100 and Drive A100 | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0304` | `FBPA_MRS_1` | FBPA | `0x00100000` on every card | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0320` | `FBPA_MRS_8` (MR8 Density) | FBPA | `0x00200000` on all 15 cards, so it is **not** the capacity restriction | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0334` | `FBPA_MRS_2` | FBPA | `0x00200019` (8 GB card), `0x002000cf` (10 GB card and A100 40 GB) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0338` | `FBPA_MRS_WL_RL` | FBPA | `0x003000eb` (8 GB card), `0x003000ea` (10 GB card) | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a038c` | `FBPA_HBM_CFG0` | FBPA | `0x000000a7` on 170HX and A100, `0x000000a6` on Drive A100. Fields `dual_rank[0]`, `dual_rank_bank[1]`, `SID_VAL[11]` | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a03f0` | PLM candidate | FBPA | `0xffffff8f`; survey only, what it guards is **not documented** | [26-register PLM survey](../unlock/register-reference.md#the-26-register-plm-survey) |
| `0x009a0470` | `FBPA_ECC_CTRL` | FBPA | reads `0` with `MASTER_EN` read-only. ECC is fused off with no known lever | [ECC](../frontier/ecc.md) |
| `0x009a0554` | PLM candidate | FBPA | `0xffffffcf`; survey only, **not documented** | [26-register PLM survey](../unlock/register-reference.md#the-26-register-plm-survey) |
| `0x009a0838` / `0x009a083c` | `FBPA_VEND_ID_C0` / `C1` | FBPA | `0x00000000` on all 15 cards, so the HBM vendor ID is not exposed here | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a0974` | `FBPA_TRAINING_STATUS` | FBPA | `0x00000000` = FINISHED. SUBP0 in [1:0], SUBP1 in [3:2]; value 2 means ERROR | [Troubleshooting](../procedures/troubleshooting.md) |
| `0x009a0bfc` | PLM candidate | FBPA | reads `0x00000000`; survey only, **not documented** | [26-register PLM survey](../unlock/register-reference.md#the-26-register-plm-survey) |
| `0x009a3cb4` / `0x009a3cb8` / `0x009a3cbc` | `I1500_INSTR` / `MODE` / `DATA` | FBPA | IEEE 1500 HBM test port; `0x0000000f` / `0x00000008` / `0x40000000` | [Memory subsystem](../hardware/memory-subsystem.md) |
| `0x009a3cc0` / `0x009a3cc4` / `0x009a3cc8` | `I1500_SHADOW_WIR` / `WDR` / `STATUS` | FBPA | `0x000000f0` read-only / per-die / `0x00000000` idle. `WDR` reads `0x8000f000` on an 8 GB card and `0x8273ff83` on a 10 GB card | [Memory subsystem](../hardware/memory-subsystem.md) |

> [!NOTE]
> **Registers that look adjacent but are not related**
>
> `0x00823800` and `0x00823804` are four bytes apart and do different things:
> `0x00823800` is `FEAT_OVR_ECC_PLM` and `0x00823804` is the `FEAT_OVR_PLM` that gates the
> compute unlock. Likewise `0x00820c14` is FBIO status and `0x00820c18` is FBPA status. Both
> pairs have been swapped in circulated notes.

---

## Payload offsets

These are **not BAR0 addresses.** They are byte offsets into the `0x0000f800`-byte
(63,488-byte) fake signature buffer that the driver hands the Booter, which DMAs it to SEC2
DMEM `0x0800`. Therefore:

> [!NOTE]
> **The conversion rule**
>
> **DMEM address = payload offset + `0x800`.** The buffer maps 1:1 onto DMEM
> `0x0800`..`0xffff`, because `0x0800 + 0xf800 = 0x10000`, exactly the top of the 64 KB DMEM.

The whole buffer is first filled with the dword `0x000004a7`, then exactly 24 slots are
overwritten. Every value below is read directly out of
`_kgspSec2PostblTimingFillPayload()` in `0001-sec2-postbl-plm-ss-cfg.patch`, and the table is
**byte-identical in shipping `master` and in all twelve archived branches**. Two of the slots
are the arguments: the address you want written and the value you want written. Everything else
is the ROP tail.

| Payload offset | DMEM address | Value | Role |
|---|---|---|---|
| (all) | `0x0800`-`0xffff` | `0x000004a7` | background fill dword; why this constant is **not documented** |
| `0x1100` | `0x1900` | `0x00000007` | **not documented** |
| `0x5b40` | `0x6340` | `0xc0deca7e` | **the fake canary written into the stack-guard global.** The address `0x6340` is the load-bearing fact; the value is arbitrary as long as it matches every saved copy |
| `0xf754` | `0xff54` | *writeValue* | the value argument, lowest tail slot |
| `0xf758` | `0xff58` | `0xc0deca7e` | saved-canary slot |
| `0xf75c` | `0xff5c` | `0x00000cbd` | Falcon IMEM address; role **not documented** |
| `0xf76c` | `0xff6c` | *writeAddr* | the BAR0 address argument |
| `0xf774` | `0xff74` | `0x00001fbd` | IMEM address in the "elevator" gadget family |
| `0xf780` | `0xff80` | `0x00000000` | **not documented** |
| `0xf788` | `0xff88` | `0x000010aa` | **the BAR0-master write gadget, `reg_write_indirect`.** This is the slot that makes the whole exploit a write primitive |
| `0xf78c` | `0xff8c` | `0x0000815a` | IMEM address; role **not documented** |
| `0xf790` | `0xff90` | `0x00008e18` | IMEM address; role **not documented** |
| `0xf794` | `0xff94` | `0xc0deca7e` | saved-canary slot |
| `0xf798` | `0xff98` | `0x0000815a` | second copy of the same IMEM address |
| `0xf79c` | `0xff9c` | `0x00000000` | **not documented** |
| `0xf7a0` | `0xffa0` | `0xc0deca7e` | saved-canary slot |
| `0xf7a4` | `0xffa4` | `0x00001fbd` | second copy |
| `0xf7b0` | `0xffb0` | `0x0000ffbc` | IMEM address; role **not documented** |
| `0xf7b8` | `0xffb8` | `0x0000582d` | IMEM address; role **not documented** |
| `0xf7c4` | `0xffc4` | `0xc0deca7e` | saved-canary slot |
| `0xf7c8` | `0xffc8` | `0x00000cbd` | second copy |
| `0xf7d8` | `0xffd8` | `0x00000003` | **not documented** |
| `0xf7e0` | `0xffe0` | `0x00001fbd` | third copy |
| `0xf7f4` | `0xfff4` | `0x00000ccb` | `regtable_rw_indexed`, and an open problem: it indexes the very descriptor tables the payload smashes, yet the unlock works |
| `0xf7f8` | `0xfff8` | `0x00007f2f` | outermost slot; role **not documented** |

The canary `0xc0deca7e` appears exactly five times per copy: at `0x5b40` and at payload offsets
`0xf758`, `0xf794`, `0xf7a0`, `0xf7c4`.

> [!NOTE]
> **Open problem: the unexplained payload constants**
>
> Fifteen of the twenty-four slots have no confirmed role, carrying ten distinct constants
> between them. The ROP write-ups name a
> neighbouring gadget family (`0x1fb9`, `0x1fca`, `0x814e`, `0x8173`, `0x7f82`), so these are
> plausibly the same tail translated, but nobody has walked the annotated disassembly to
> confirm it. See [The ROP chain](../unlock/rop-chain.md) and
> [Open questions](../frontier/open-questions.md).

### Load-bearing DMEM addresses referenced by the payload

Same space as the right-hand column above, given here as DMEM addresses because that is how the
disassembly refers to them.

| DMEM address | Meaning |
|---|---|
| `0x0100` and below | nothing is allocated here, which is what killed the "mega-ROP staged in low DMEM" idea |
| `0x0530` | DMA and engine config descriptor |
| `0x0600` | `WprMeta`, a 256-byte structure |
| `0x06fc` | where the Booter stores `0xa0a0a0a0` on the `r4 == 0` branch. **Unrelated to the `0x001fa824` / `0x001fa828` WPR2 registers**, despite the coincidental digits |
| `0x0800` | base of the DMA'd signature buffer, that is, payload offset 0 |
| `0x103c` onward | crypto-session descriptor |
| `0x2383` and `0x8e08` | register descriptor tables, which the payload linearly smashes |
| `0x6340` | the stack-canary global, 25408 decimal |
| `0x8700` | end of Booter code and data |
| `0xffec` | the slot that feeds `main`'s exit status and decides whether `secure_teardown` runs |

### Payload sizes and canaries across variants

| Variant | Buffer size | DMA base | Canary value |
|---|---|---|---|
| Shipping `master` and all 12 branches | `0x0000f800` = 63,488 B | `0x0800` | `0xc0deca7e` |
| Clean-room ROP write-ups | `0x0000f800` | `0x0800` | `0xfaceb13d` |
| Superseded `builder.py` / `patcher.py` | `0x0000f700` = 63,232 B | `0x0900` | `0xdead2c20` at `0x2c20` |

The `0x0900` DMA base of the superseded tooling is why one archived message gives the canary
address as `0x6440`: `0x5b40 + 0x900 = 0x6440`. On the shipping path the base is `0x0800` and
the canary lives at `0x6340`.

---

## Related pages

- [Register reference](../unlock/register-reference.md), the explanatory companion to this index
- [Privilege level masks](../unlock/privilege-level-masks.md) for the PLM nibble encoding
- [How the unlock works](../unlock/how-it-works.md) and [Driver patches](../unlock/driver-patches.md)
- [Memory geometry](../unlock/memory-geometry.md), [Compute throttle](../unlock/compute-throttle.md),
  [PCIe Gen2](../unlock/pcie-gen2.md)
- [Falcon and Booter](../unlock/falcon-and-booter.md) and [The ROP chain](../unlock/rop-chain.md)
- [Fuses and OTP](../hardware/fuses-and-otp.md),
  [Memory subsystem](../hardware/memory-subsystem.md),
  [PCIe subsystem](../hardware/pcie-subsystem.md)
- [Glossary](../start/glossary.md) for PLM, FLR, WPR, FBPA, LMR, AON, HS, PL0/PL3
- [Artifacts](artifacts.md) and [External sources](external-sources.md)
