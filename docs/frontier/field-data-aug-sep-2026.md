# Field data settling open questions (Aug–Sep 2026, one Haswell-EP server rig)

> [!NOTE]
> **Single-rig field report, ASUS Z10PE-D8 WS (C612, Xeon E5 v3/Haswell-EP), BIOS 4101.** Two
> cards: `10de:20c2` (8 GB Hynix → 64 GB) and `10de:2082` (10 GB Samsung → 40 GB), both on CPU2
> root ports (81:00.0, 82:00.0). Driver stack: cmpunlocker v0.3 (tag `f51da03`) → master `fe53796`,
> nvidia-open 610.43.03, kernel 7.0.0-31-generic. Everything below was measured on this rig with
> the named instrument; nothing is inferred from logs alone. This page exists to close open
> questions [1.2](open-questions.md#12-rmpcielinkspeed0x1-or-0x2), [2.12](open-questions.md#212-recoverable-fbps),
> [2.13](open-questions.md#213-iommu-and-the-gen2-reliability-story), and
> [2.14](open-questions.md#214-reboot-persistence-of-gen2), and to weigh in on the
> [transient-window dispute](../unlock/pcie-gen2.md) on the Gen2 page.

## 1. Q1.2 — `RMPcieLinkSpeed=0x1` vs `0x2`: no difference; neither is sufficient

The A/B the question asked for, run **after** fixing an initramfs key-delivery trap that
invalidates every earlier "key doesn't work" observation made from a systemd-era hammer:

- The modprobe conf (`RmForceEnableGen2=1;RMPcieLinkSpeed=…`) must be **inside the rebuilt
  initramfs** when the module loads from the initrd (ours loads at T+1 s). A conf file on the
  root filesystem reaches nothing in that case. Check:
  `lsinitramfs /boot/initrd.img-$(uname -r) | grep cmp-pcie-gen2`.
- With keys verified delivered via `/proc/driver/nvidia/params`, two boots: `0x1` (2026-08-31
  00:25) and `0x2` (2026-08-31 00:43). Both boots: LnkCap `0x00456102`, phases A+B land, full
  hammer coverage (600 attempts/card, 0/1200 each boot), **LnkSta `0x1041` (Gen1) both times.**

Verdict: on this platform class the `0x1`-vs-`0x2` spelling is moot — the retrain is refused for
a reason the key does not control (see §3). Full A/B detail: upstream issue
[amoghmunikote/cmpunlocker#2… field report of 2026-08-31](https://github.com/Consensus-Protocol/cmp170hx/issues/2).

## 2. Q2.13/Q2.14 — the Gen2 failure class this rig hit: a ~0.4 s boot window

Three wrong "hardware wall" verdicts on this rig preceded the real root cause; the failure
signature of a timing wall and a hardware wall are identical (every fix "correctly applied",
100% failure).

**Mechanism (verified 2026-09-08):** the Gen2 advertisement window opens when the patched
module's booter writes land — on an initramfs-loaded module that is **~T+6 s** — and **closes
~0.4 s later**, before any systemd userspace unit can run. The driver's own probe retrain also
fires after the window closes. Corroboration on other rigs: the same failure signature
(0/600 hammer, LnkSta pinned `0x1041`) with stock tooling, and Gen2 **working** on an older
C602 board (ASUS Z9PE-D8 WS) once a watcher catches the flip inside the window — plus a
same-card, same-driver (2082 / 610.43.03) confirmation of a catch at T+6.262 s on Debian with
kernel 6.8.

**Fix shipped on this rig (kmike's closed PR amoghmunikote/cmpunlocker#35, staged manually):**
an initramfs-stage watcher (`tools/watch-setup.sh`) that retrain-hammers during the window,
before udev. Result, 2026-09-08, one warm reboot: **LnkSta `0x1042` (Gen2 x4) on both cards**,
measured H2D 1.51–1.52 GB/s, D2H 1.56 GB/s (2 GiB pinned, median of 5, checksummed; was
~0.85 GB/s → 1.8×).

Consequences for the open questions:

- **Q2.14 (reboot persistence):** the "2,2 after install, 1,1 after reboot" signature is exactly
  this window miss — the installer's retrain lands inside the window, the boot-time retrain
  doesn't. Persistence requires a **boot-time catch** (initramfs watcher), not just the patched
  module being loaded. The watcher is now a permanent boot dependency; link state does not
  survive power cycles.
- **Q2.13 (IOMMU):** on this rig `intel_iommu=on iommu=pt` was in place across all failures and
  the eventual success; IOMMU mode was never the discriminator. The discriminator was when the
  retrain fires relative to the window.
- **Measure only endpoints:** `nvidia-smi --query-gpu=pcie.link.gen.current/max` **reports 1
  even after a successful Gen2 catch** on this stack (NVML reads through the RM, which never saw
  the early-boot retrain). Judge Gen2 by config-space LnkSta only:
  `setpci -s <bdf> CAP_EXP+12.w` → `0x1042` = Gen2 x4, `0x1041` = Gen1. The `0008` retrain
  dmesg success predicate (open question 0.4) is also unreliable — the "retrain completed
  without Gen2 link" line prints on working rigs.

## 3. Weigh-in on the transient-window dispute

The steady-state dump reading `LnkCap2 = 0x00000006` after boot does not contradict the
transient model — it is the **post-window residue**. On this rig the driver log shows
`CAP=0x00456102` at T+8 s (window open) while NVML reports max=1 hours later: the advertisement
flips up during the window and is re-clamped after it closes, but a link that trained at Gen2
while the window was open stays trained. A timestamped LnkCap2/LnkSta poll from early boot
through 60 s would show the flip and the re-clamp as separate events; on this rig the window
was T+6.0→T+6.4 s approximately.

## 4. Q2.12 — Recoverable FBPs: fuse-map answer from gpcprobe (2026-09-23)

Eight-register fuse dump via [gpcprobe](https://github.com/matstatman/gpcprobe) on both cards,
read through `/dev/mem` at BAR0 (host reads of fuse registers are permitted; PLMs gate writes):

- **`10de:20c2` (8 GB Hynix):** `FBP_DISABLE ≠ FBP_DEFECTIVE` — FBP 6, 11 defective; FBP 1, 4
  **disabled but not defective** (the case Q2.12 asked for: 96 GiB theoretical if revived —
  2 full + 4 half stacks, 16 Gb dies; live CFG1 already reads the A100-80GB value `0x02779000`).
- **`10de:2082` (10 GB Samsung):** `DISABLE == DEFECTIVE == 0x24` — every disabled FBPA/FBP is
  also defective. 40 GB is the physical maximum; nothing revive-able, case closed.
- Both cards: GPC 0, 1, 5 disabled-not-defective (the 70-SM SKU cut).

**Why revival is walled anyway:** on ALL 170HX probed, `FUSE_EN_SW_OVERRIDE (0x820040) = 0` and
`FUSE_DIS_SW_OVR = 1` — the `CTRL_OPT_*` live-override mechanism is fused off (all `CTRL` regs
read 0, and the port is value-gated: cross-value writes are dropped at the port, never accepted
even transiently — 200-poll proof, 2026-08-16). Revival would additionally need FB-Falcon
boot-time channel retrain. So Q2.12's caveat is answered pessimistically: "disabled but not
defective" silicon is NOT recoverable by any host-visible path on stock cards.

## 5. Related corrections this rig filed elsewhere

- The Xid-31 repack class on 40 GB cards is cmpunlocker's own `late-pma.patch` registering
  WPR2/GSP-firmware pages into PMA (PR amoghmunikote/cmpunlocker#32, merged as `ed57921`):
  the fault fires only once the main pool is exhausted, which is why it looked Samsung-specific.
- Mixed-arch rigs: sm_86 (3090) triton first-use `cuModuleLoadData` races under the patched
  610.43.03 stack — field report PR amoghmunikote/cmpunlocker#36.
- 80 GB on Samsung XA2_8HI dies: address folding confirmed die-side (two experiments, Aug 14–15);
  consistent with the fuse map above. The refire chain's no-fold result on this wiki's 80 GB page
  remains unexplained on this rig's card — lottery silicon or delivery-context difference.
