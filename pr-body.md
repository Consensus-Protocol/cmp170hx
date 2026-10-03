# Field data (Aug–Sep 2026): settles Q1.2, Q2.12, Q2.13, Q2.14; weighs in on the Gen2 transient-window dispute

Single-rig field report from an ASUS Z10PE-D8 WS (C612, Xeon E5 v3/Haswell-EP), BIOS 4101 — a server-vintage platform none of the corpus-freeze rigs covered. Cards: `10de:20c2` (8 GB Hynix → 64 GB) and `10de:2082` (10 GB Samsung → 40 GB). Stack: cmpunlocker v0.3 → master `fe53796`, nvidia-open 610.43.03.

What this adds:

- **Q1.2 answered** (one platform class): after fixing an initramfs key-delivery trap (modprobe conf on the root fs reaches nothing when the module loads from the initrd at T+1s — check with `lsinitramfs`), a full A/B of `RMPcieLinkSpeed=0x1` vs `0x2` shows **no difference — both train Gen1** with keys verified in `/proc/driver/nvidia/params` and 0/1200 hammer coverage per boot.
- **Q2.14 answered** (root cause): the "2,2 after install, 1,1 after reboot" signature is a **boot-window timing miss**. The Gen2 advertisement window opens when the booter writes land (~T+6 s on an initramfs-loaded module) and **closes ~0.4 s later**, before any systemd userspace retrain fires. An initramfs-stage watcher (the kmike PR#35 class) catches it every boot — verified LnkSta `0x1042` on both cards, H2D 1.5 GB/s measured (was 0.85).
- **Q2.13 data point:** `intel_iommu=on iommu=pt` held constant across every failure and the eventual success — IOMMU was never the discriminator; window timing was. Supports the "IOMMU only matters on some chipsets" arm.
- **Transient-window dispute:** the steady-state `LnkCap2=0x6` dump is consistent with the transient model — it's post-window residue; the advertisement flips up during the window and is re-clamped after, while a link trained inside the window stays trained.
- **Q2.12 answered** (gpcprobe fuse dump, both SKUs): the 8 GB card does show `DISABLE ≠ DEFECTIVE` (FBP 1,4 disabled-only, 96 GiB theoretical), but revival is doubly walled on all 170HX: `FUSE_EN_SW_OVERRIDE=0`/`FUSE_DIS_SW_OVR=1` (live-override mechanism fused off; the port is value-gated — cross-value writes dropped, 200-poll proof) plus no host-visible FB-Falcon retrain path. The 10 GB card: `DISABLE == DEFECTIVE` — 40 GB ceiling is physical.
- **Measurement warning:** `nvidia-smi pcie.link.gen.current/max` reports 1 even after a verified Gen2 catch on this stack — NVML reads through the RM and never sees the early-boot retrain. LnkSta via `setpci` is the only honest indicator.

Also cross-references two root causes filed upstream on amoghmunikote/cmpunlocker: the Xid-31 repack class = `late-pma.patch` WPR2/PMA registration (PR #32), and the sm_86 triton module-load race on mixed-arch rigs (PR #36).

Conventions followed: no individuals named, findings attributed to dates, single-observation claims say so, measurements are instrument-backed.
