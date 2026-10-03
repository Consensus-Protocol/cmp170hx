# Field data Aug–Oct 2026 (one Haswell-EP rig): Gen2 window root cause, gpcprobe fuse map, FF-pool decode with anchors, RIR verification + corrected burn lists, 96 GB append recipe

Single-rig field reports from an ASUS Z10PE-D8 WS (C612, Xeon E5 v3/Haswell-EP), BIOS 4101. Cards: `10de:20c2` (8 GB Hynix → 64 GB) and `10de:2082` (10 GB Samsung → 40 GB). Stack: cmpunlocker v0.3 → master `fe53796`, nvidia-open 610.43.03.

What this branch adds:

**Gen2 (closes Q1.2, Q2.13, Q2.14):**
- Q1.2: after fixing an initramfs key-delivery trap, full A/B of `RMPcieLinkSpeed=0x1` vs `0x2` — no difference, both train Gen1 with keys verified delivered
- Q2.14 root cause: Gen2 advertisement window opens ~T+6 s (booter writes) and closes ~0.4 s later, before any systemd userspace retrain; initramfs-stage watcher (kmike PR#35 class) catches it every boot — verified LnkSta 0x1042 both cards, 1.5 GB/s measured
- Q2.13: `intel_iommu=on iommu=pt` constant across failure and success — window timing was the discriminator
- `nvidia-smi pcie.link.gen.current/max` reports 1 even after verified Gen2 catch; LnkSta via setpci is the only honest indicator

**Fuse architecture (2026-09/10, gpcprobe + fuse-controller dumps):**
- Q2.12: gpcprobe fuse dump — 10 GB card DISABLE==DEFECTIVE (40 GB ceiling physical); 8 GB card FBP 1,4 disabled-only (96 GiB theoretical) but volatile override paths fused off (FUSE_EN_SW_OVERRIDE=0)
- **FF-pool decode**: chain=bits[5:0], offset=bits[15:6], data=bits[31:16] — anchored to DEVIDA/B (0x20c2/0x2082), OPT_FBP_DISABLE (0x852/0x24), OPT_GPC_DISABLE (0x23); 145 records, lossless re-encode, artifacts in `docs/frontier/artifacts/`
- **RIR macro verified**: slot 15 = 0x0923 on our Hynix, substitution live in read path (row 18 bit 8 vs unsubstituted row 19); redirect target row 287 offset 39→47 with last-writer-wins cover confirmed
- **⚠️ CORRECTION**: the circulated RIR burn list [2,6,7,10,12,13] is wrong — decodes to a different redirect (row 123). Correct bits: [2,7,9,10,15] → 0x8FA7
- **Derived 64→96 GB recipe** (Hynix 8 GB, FBP mask 0x852): FF append `0x420004C1` (chain 1 / offset 19) clears disable bits 1,4 → +32 GB theoretical. Gated on FF-pool analog-write evidence, 96 GB geometry values, and FB-Falcon retrain behavior — published for verification and trade against primary burn evidence
- Raw 512-row dumps for both SKUs published as open data

Also cross-references upstream root causes filed on amoghmunikote/cmpunlocker: Xid-31 = late-pma WPR2/PMA bug (PR #32), sm_86 triton module-load race (PR #36).

Conventions followed: no individuals named, findings attributed to dates, single-observation claims say so, measurements instrument-backed.
