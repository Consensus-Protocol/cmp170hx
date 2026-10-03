# MIG Test Report — CMP 170HX (2nd Independent Data Point)

## Setup

- **GPU under test:** 1× NVIDIA CMP 170HX (GA100, unlocked to 64 GB); MIG exercised on this card only. A second GPU (RTX 3060) shared the host but was idle and uninvolved.
- **NVIDIA Driver:** 610.43.03 (open-gpu-kernel-modules)
- **cmpunlocker (base):** [amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker) — the 10-patch `master` set as it stood ~2026-09-01, *before* `cmp-sku-mask.patch` was added upstream (today's `master` carries 11). `PATCH_ORDER`: `sec2-postbl-plm-ss-cfg · booter-verify · late-pma · bar0-pramin-clamp · ce-scrub-workarounds · persistent-sw-state · pcie-gen2 · pcie-gen2-probe-retrain · name-string · bar1-resize-unlock`. Local build — the tree is a copy, **not** a git checkout, so no upstream commit is pinned in it. Auto-profile `8gb` → 64 GB unlock (PCI `10de:20c2`). Base `nvidia.ko` = 41 625 960 B.
- **MIG Patch:** [mig-unlock.patch](https://github.com/amoghmunikote/cmpunlocker/blob/MIG/driver/patches/mig-unlock.patch) (MIG-Branch, commit `86b238f` — "Widen the MIG compute partition from 14 SMs to the whole GPU")
- **OS:** Debian 13, Kernel 6.12.107+deb13-amd64
- **Workload run on the instance:** two single-file CUDA programs, built for `sm_80` with CUDA 13.1 (`nvcc -arch=sm_80 …`, cuBLAS linked with `-lcublas`), each pinned to the MIG compute instance via `CUDA_VISIBLE_DEVICES=<MIG-UUID>`:
  - `sgemm` — cuBLAS `cublasSgemm`, square FP32 GEMM N=8192, 30 timed iterations; GFLOP/s = 2·N³·iters ÷ elapsed.
  - `rt` — a 512 MiB host→device→host `cudaMemcpy` round-trip with a full byte-compare verify.

**Patch applies cleanly to 610.43.03** (`patch -p1 --dry-run`): all 3 hunks succeed (2 & 3 at offset −31, no fuzz). Built via cmpunlocker `PATCH_ORDER`; **the two builds differ by exactly one entry — `mig-unlock.patch` appended last to the 10-patch base, nothing else** — `nvidia.ko` = 41 627 368 B (patched) vs. 41 625 960 B (base). No brick; driver loads cleanly, 64 GB intact, both across a warm reboot and a cold power-cycle.

## Headline finding

The patch does **not** change profile listing or instance creation — **both work on the cmpunlocker base without `mig-unlock.patch`**. The measured A/B difference is **FP32 SGEMM throughput on the instance**:

| Driver | `mig -lgip` (GI listing) | create `1g.64gb` instance | **cuBLAS SGEMM (8192³ FP32) on the instance** |
|---|---|---|---|
| **base (unpatched)** | lists `1g.64gb` | succeeds | **1848 GFLOP/s** |
| **+ mig-unlock.patch** | lists `1g.64gb` (unchanged) | succeeds | **10225 GFLOP/s** |

So on this driver (610.43.03) the patch raises FP32 SGEMM under MIG by **~5.5×** — **consistent with the patch widening the effective compute-instance capacity**, as the commit describes ("14 SMs → the whole GPU"; the commit's own count is 14 → 56 SMs). This is **not** an SM count: `-lgip`/`Shared MP count` cosmetically report 70 SM regardless and are not evidence of the real partition width — only the throughput delta is measured here, and only for FP32.

**Not established by this test:** the actual active-SM count (needs the CUDA `multiProcessorCount` device attribute and/or `-lci` compute-instance profile data), clocks/power/temperature, and repeated runs. It also does **not** address the separate open **INT8/IMMA throughput** gate (a different question — library vs. raw MMA — not FP32-SGEMM).

## Test Results

*The commands and outputs below are transcribed from the run (the GPU UUID is redacted; repeated rejection lines are abbreviated as `→ ...`). Enable / list / create / profile-rejection behave identically on the base and patched drivers — the only measured difference in the A/B is the SGEMM throughput.*

### MIG enable + mode
```
$ sudo nvidia-smi -i 0 -mig 1
Enabled MIG Mode for GPU 00000000:81:00.0
All done.

$ nvidia-smi -i 0 --query-gpu=name,mig.mode.current,mig.mode.pending --format=csv,noheader
NVIDIA CMP 170HX, Enabled, Enabled
```

### GPU-instance profile list (`mig -lgip`) — exactly one profile, whole-GPU
```
+-------------------------------------------------------------------------------+
| GPU instance profiles:                                                        |
| GPU   Name               ID    Instances   Memory     P2P    SM    DEC   ENC  |
|                                Free/Total   GiB              CE    JPEG  OFA  |
|===============================================================================|
|   0  MIG 1g.64gb          0     1/1        63.00      No     70     0     0   |
|                                                               5     0     0   |
+-------------------------------------------------------------------------------+
```

### Create GPU instance + compute instance, then list it
```
$ sudo nvidia-smi mig -cgi 1g.64gb -C
Successfully created GPU instance ID  0 on GPU  0 using profile MIG 1g.64gb (ID  0)
Successfully created compute instance ID  0 on GPU  0 GPU instance ID  0 using profile MIG 1g.64gb (ID  0)

$ sudo nvidia-smi mig -lgi
|   0  MIG 1g.64gb            0        0          0:8     |     (GPU instance; placement start:size = 0:8)
```

### Compute throughput — the real delta (cuBLAS SGEMM 8192³ FP32 ×30, on the MIG instance)
```
base (unpatched):   17847 ms -> 1848 GFLOP/s
+ mig-unlock.patch:  3226 ms -> 10225 GFLOP/s    (~5.5x)
```
Same profile (`1g.64gb`), same card/host, single A/B difference = the patch. Process confirmed running **inside the instance** (patched run) — `nvidia-smi` Processes table:
```
|  GPU   GI   CI      PID   Type   Process name        GPU Memory |
|    0    0    0     4338      C   ./sgemm                1006MiB |
```
**The base number is not a warm-reboot artifact:** after a cold power-cycle, on a verified-unpatched driver (the `mig-unlock.patch` log signature is absent from `dmesg`), the base instance again creates and the same SGEMM runs at **1850 GFLOP/s** — matching the 1848 above:
```
$ CUDA_VISIBLE_DEVICES=<base-cold MIG dev> ./sgemm
SGEMM N=8192 x30: 17831.9 ms -> 1850 GFLOP/s
```
(Ref: full A100 ~19.5 TFLOP/s FP32; CMP is a cut-down GA100. FP32-SGEMM only; not an SM count or general MIG-performance claim.)

### 512 MiB memory round-trip (host→device→host, byte-verify, on the MIG device)
```
512 MiB round-trip (host->device->host, verify): OK
```

### Partial / multi-instance profiles — not available (base and patched)
Only `1g.64gb` (whole GPU) is in the RM profile list; every standard A100 profile is rejected. First rejection verbatim, the rest abbreviated:
```
$ sudo nvidia-smi mig -cgi 7g.40gb -C
Unable to create a GPU instance on GPU  0 using profile 7g.40gb: Invalid Argument
Failed to create GPU instances: Invalid Argument
$ sudo nvidia-smi mig -cgi 4g.20gb -C    → ... using profile 4g.20gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 3g.20gb -C    → ... using profile 3g.20gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 2g.10gb -C    → ... using profile 2g.10gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 1g.10gb -C    → ... using profile 1g.10gb: Invalid Argument
$ sudo nvidia-smi mig -cgi 1g.5gb  -C    → ... using profile 1g.5gb:  Invalid Argument
$ sudo nvidia-smi mig -cgi 5 -C          → ... using profile 5:  Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 9 -C          → ... using profile 9:  Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 14 -C         → ... using profile 14: Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 19 -C         → ... using profile 19: Invalid Argument   (by ID)
$ sudo nvidia-smi mig -cgi 9,3g.20gb -C  → ... using profile 9:  Invalid Argument   (exact wiki attempt)
```

### `nvidia-smi -q` (MIG section, instance active) — verbatim
```
    MIG Mode
        Current                                        : Enabled
        Pending                                        : Enabled
    MIG Device
        Index                                          : 0
        GPU Instance ID                                : 0
        Compute Instance ID                            : 0
        Device Attributes
            Shared
                Multiprocessor count                   : 70
                Copy Engine count                      : 5
                Encoder count                          : 0
                Decoder count                          : 0
                OFA count                              : 0
                JPG count                              : 0
        Shared FB Memory Usage
            Total                                      : 64912 MiB
            Reserved                                   : 0 MiB
            Used                                       : 1 MiB
            Free                                       : 64912 MiB
        Shared BAR1 Memory
            Total                                      : 65536 MiB
            Used                                       : 0 MiB
            Free                                       : 65535 MiB
GPU UUID : GPU-<redacted>
```

## Observations

- MIG **enables and holds** (through create/destroy and driver swaps); returned cleanly to non-MIG via `-mig 0`.
- **`mig-unlock.patch`'s observable effect here is compute throughput, not enabling MIG or creating the instance** (the base already does both). It raises FP32-SGEMM under MIG **~5.5× (1848 → 10225 GFLOP/s)**, consistent with widened compute-instance capacity (commit: "14 SMs → whole GPU"). The active-SM count is **not** established — `-lgip`/`Shared MP count` report 70 SM with and without the patch and are not evidence of the real partition; only the throughput delta is measured.
- **No sub-partitioning / multiple instances** — the RM GA100 GPU-instance profile list contains only the whole-GPU `1g.64gb` entry; all standard A100 profiles return `Invalid Argument` (base and patched alike).
- No brick / no persistent hardware writes; fully reversible (module restore + `-mig 0` + reboot). Verified twice (warm revert, then cold power-cycle back to the base driver).

## Root cause: evidence for a hardware-enumeration limit (syspipe-fuse attribution provisional)

> [!NOTE]
> This answers Next Step #2 and bears on open-questions §1.5 ("fused off vs. not programmed"). Measured on both 170HX units with driver **610.57.04** (open-gpu-kernel-modules) built through cmpunlocker (amoghmunikote master `6c442ee`, 74 SM) plus three BAR1-P2P patches from bayley/cmpunlocker, not with the 610.43.03 build above. `0x008203a4` sits in the fuse-shadow range; the role of `0x00820e30` is inferred. The values were not compared across drivers.

Per static analysis of the GSP-RM (`gsp_tu10x.bin`, 610.57.04), the physical RM builds the GA100 GPU-instance profile list from the hardware device-info table (PTOP `DEVICE_INFO2`), not from a patchable RM profile array. The measurements below show the limit in three layers.

**1. Only one GRAPHICS engine is enumerated.** Instrumenting the kernel RM (`gpuConstructDeviceInfoTable_FWCLIENT` in `gpu_gspclient.c`, which receives the `NV2080_CTRL_CMD_INTERNAL_GET_DEVICE_INFO_TABLE` reply from the GSP) dumps the full table: **15 entries, exactly one GRAPHICS engine** (type `0x0`, `devicePriBase 0x400000` = PGRAPH, GR0), plus 10 copy engines (type `0x13`) and no video engines. A raw, read-only BAR0 dump of `NV_PTOP_DEVICE_INFO2` (`0x22800`, `NUM_ROWS=88`, up to 3 rows per device), read directly from the hardware and not through the GSP, independently contains the GR0 type-0 record and no other non-zero type-0 record. It matches 14 of the 15 GSP records; the GSP-only type-`0x34` record (not an engine, `devicePriBase 0`) has no PTOP counterpart. The dump therefore supports a single exposed GRAPHICS record but does not establish that the GSP performs no filtering. Directly after GR0 follow 21 zeroed rows, which we read as seven absent three-row GR slots (GR1..GR7); that layout reading is an interpretation.

**2. `MAX_MIG_ENGINES` is not the gate.** Instrumenting `kgraphicsLoadStaticInfo_KERNEL` (`kernel_graphics.c`) shows the GR static info reporting `MAX_MIG_ENGINES = 8` and `MAX_PARTITIONABLE_GPCS = 8` (`LITTER_NUM_GPCS = 8`), i.e. the GA100 architectural maxima. Per the static GSP-RM analysis, the usable number of GPU instances follows the number of enumerated GRAPHICS engines, which is 1, so only the whole-GPU `1g.64gb` instance is offered. NVML agrees: `nvmlDeviceGetGpuInstanceProfileInfo` returns only index 0 (`1_SLICE`, 70 SM, 64512 MB); indices 1..14 are "Not Supported". Enabling MIG does not change this: with MIG on, `-lgip` still lists only `1g.64gb`, `-cgi 19,19` fails with `Invalid Argument`, and `-cgi 0 -C` creates the single whole-GPU instance.

**3. A syspipe-shaped mask reads `0xfe` and does not take host writes.** Two registers read **`0xfe`** on both units (read-only BAR0 survey of `0x00820000..0x00821000`): `0x008203a4` in the floorsweep-fuse readback block and `0x00820e30`. By bit pattern and position we read them as the chiplet syspipe / SMC-engine disable mask and a possible applied copy (bits 1..7 set = syspipes 1..7 disabled, only syspipe 0 active); the names and the syspipe-to-GR mapping are inferred, not taken from a header. On the GA106 control card (RTX 3060) `0x008203a4` reads `0x0` and `0x00820e30` is not decoded (`0xbadf5040`); no A100 reading of these addresses exists yet. A write-probe on **one card (81:00.0), GPUs idle**, wrote `0xfc` (clear bit 1) to **both registers** from the host through BAR0 and read back `0xfe`: the writes did not latch, there was no Xid and no log noise, and the original value was restored. `RECONFIG_PLM 0x008200d8` read `0xffffffff` at the time, but it is not established that this PLM governs these two addresses. The result is consistent with `FUSE_DIS_SW_OVR 0x00820084 = 1`, which reads `1` on all surveyed cards (incl. the RTX 3060), so it corroborates rather than proves anything CMP-specific. Unlike the TPC floorsweep, which has a writable per-GPC override (`0x00820a40`) that the unlock uses, **no writable override was found for this mask**.

**MIG mode applies a stricter TPC mask.** Enabling MIG reboots the GSP on that GPU, and the unlock's SM-reconfig runs again. On 01:00.0 (`OPT_GPC_DISABLE = 0x23`, GPCs 0, 1 and 5 fused off) the per-GPC TPC status values that the unlock logs at GSP boot (registers `0x00820c38 + gpc*4`) were:

| GPC | 2 | 3 | 4 | 6 | 7 |
|---|---|---|---|---|---|
| MIG off (after unlock reconfig) | `0x01` | `0x00` | `0x00` | `0x01` | `0x01` |
| MIG on | `0x03` | `0x03` | `0x03` | `0x07` | `0x07` |

Interpreting these status words as TPC disable masks: with MIG off three of 40 TPCs are disabled, which matches the 74 SM that CUDA reports; with MIG on twelve are disabled, i.e. 28 TPCs or 56 SM, and the unlock's override write (`0xff` to `0x00820a40 + gpc*4`) does not change the status. The SM count inside the MIG instance was not measured (`-lgip` reports 70 SM either way). After `-mig 0` the masks return to the MIG-off values and CUDA again reports 74 SM.

**Conclusion.** These measurements place the one-profile limit below the host RM: the direct PTOP table exposes GR0 as its only non-zero GRAPHICS record, and the GSP reports the same single GRAPHICS engine. Reading the 21 zero rows after GR0 as GR1..GR7 remains an interpretation. The results are consistent with a fused syspipe configuration (`0x008203a4 = 0xfe`, host writes do not latch), but they do not yet prove that `0x008203a4` is the causal fuse or that it maps one-to-one to GR engines. None of the tested settings changed profile availability. With MIG disabled, adding `RMDebugSyspipeOverride=0x1` (read live by the 610.57.04 GSP-RM; per the static analysis it only selects among existing syspipes) did not alter the profile list or the device-info table. With the key present, the MIG-enabled run still exposed one profile; no MIG-enabled run without the key was captured, so an isolated MIG-on effect is not established. The host BAR0 writes did not latch. This is not a general exclusion of RM or GSP modifications. If the attribution holds, partitioning is **fused off rather than "not programmed"**, and since fuse bits are one-time-programmable there is no non-destructive way back; the cmpunlocker author reaches the same assessment ("it's a hard fuse"). An A100 reading of `0x008203a4` and `0x00820e30` would settle the attribution.

*Method and artifacts: GSP-RM static analysis (Ghidra 12.1.4) of `gsp_tu10x.bin` from 610.57.04, SHA-256 `d157e3b7dd5da2ca8d1ccb6ca98958f9e35d10a9ef7326277ebac133e4b0d1a7` (`ucodes_tu10x.bin`: `dcbdf512ab09b7f6d946e7f415bcfbc9137f5db1cf0c3c576f4b34b5275acb38`); kernel-RM instrumentation (device-info table, GR static info), loaded once per run and rolled back; read-only BAR0 dumps of PTOP and the fuse block on both units; one write-probe on one unit (81:00.0); MIG enable/disable on 01:00.0. All on 2× CMP 170HX, VBIOS 92.00.6D.00.0A.*

## References

- [Commit 86b238f — "Widen the MIG compute partition from 14 SMs to the whole GPU"](https://github.com/amoghmunikote/cmpunlocker/commit/86b238f0974aedf9bf61463c03d89ece4415d719)
- [mig-unlock.patch](https://github.com/amoghmunikote/cmpunlocker/blob/MIG/driver/patches/mig-unlock.patch)
- [Consensus-Protocol Wiki: Frontier-Open-Questions §1.5](https://github.com/Consensus-Protocol/cmp170hx/blob/main/docs/frontier/open-questions.md) · [Status-Board](https://github.com/Consensus-Protocol/cmp170hx/blob/main/docs/frontier/status-board.md)
- [abobasixseven/unlock-cmp-170hx — Issue #1](https://github.com/abobasixseven/unlock-cmp-170hx/issues/1)

## Next Steps (per Wiki)

1. Repeat on a second card — done here (CMP 170HX, driver 610.43.03). The patch applies, MIG enables, `1g.64gb` survives a reboot, and FP32 SGEMM under MIG rises ~5.5× (1848 to 10225 GFLOP/s), consistent with widened compute-instance capacity. Still to capture: the CUDA active-SM count and `-lci` profile data. Note that MIG mode applies a stricter TPC mask (see [Root cause](#root-cause-evidence-for-a-hardware-enumeration-limit-syspipe-fuse-attribution-provisional)), so the instance's SM count needs to be measured through the MIG device.
2. **Find where RM builds the GA100 GPU-instance profile list so more profiles can be added** — answered in [Root cause](#root-cause-evidence-for-a-hardware-enumeration-limit-syspipe-fuse-attribution-provisional): the list follows the hardware device-info table, which enumerates only GR0. Remaining: validate the syspipe-mask attribution with an A100 reading of `0x008203a4` and `0x00820e30`.
3. If the enable holds, open the pull request (maintainer has offered to merge). — Enable holds across create/destroy and reboots on 610.43.03.
