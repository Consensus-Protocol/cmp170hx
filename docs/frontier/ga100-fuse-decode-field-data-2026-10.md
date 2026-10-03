# GA100 FF-Pool Decode, RIR Verification, and Corrected Burn Lists (field data, 2026-10-02)

Single-rig field report: ASUS Z10PE-D8 WS (C612/Haswell-EP), two cards — `10de:20c2`
(8 GB Hynix, 64 GB unlocked) at 81:00.0 and `10de:2082` (10 GB Samsung, 40 GB unlocked) at
82:00.0. cmpunlocker master `fe53796`, nvidia-open 610.43.03. Dumped via the fuse
controller command interface (CMD_READ only; RIR macro via `DEBUGCTRL 0x820060` RWL=1,
FUSEADDR mod 8, DEBUGCTRL cleared + readback-verified after). Raw dumps + decoded instruction maps
published alongside this page in `artifacts/`.

## 1. FF instruction layout — decoded and anchored

```
chain  = word[5:0]
offset = word[15:6]
data   = word[31:16]
```

Verified anchors (decoded from our dumps vs independent OPT-shadow reads):

| Field | Source | Hynix | Samsung | Matches |
|---|---|---|---|---|
| DEVIDA | chain 2 / off 37, data bit 1 | `0x20c2` | `0x2082` | PCI device IDs |
| DEVIDB | chain 2 / off 38 | `0x20c2` | `0x20c2` | — |
| OPT_FBP_DISABLE | chain 1 / off 19, `(data>>3)&0xfff` | `0x852` | `0x024` | shadow reads |
| OPT_FBP_DEFECTIVE | hard rows 26/27, `(row>>16)&0xfff` | `0x840` | `0x024` | shadow reads |
| OPT_GPC_DISABLE | chain 1 / off 8, `(data>>3)&0xff` | `0x23` | `0x23` | shadow reads |

Layout scored 88/100 against alternatives (cross-card key alignment 69/76). Lossless
re-encode of all 145 records on both cards. Full per-instruction map available.

**Note:** a 10 GB card whose FF pool decodes to 14 chains / 103 instructions does not match
either of our dumps (71 and 74 nonzero instructions, chains 1/2 dominant + traces). If your
dump has 103 instructions, your FF pool boundaries or card revision differ — worth comparing
directly.

## 2. RIR macro — your claims verified, with one correction

- **Slot 15 = `0x0923`, other 15 slots `0xFFFF`**: confirmed exactly on our Hynix card.
- **The substitution is live in the read path**: our Hynix main row 18 reads `0xfcc805f9`
  while its unsubstituted twin row 19 reads `0xfcc804f9` — differing in exactly bit 8, the
  bit the record forces (addr 584 = row 18, bit 8). Samsung card: RIR macro all-zero, rows
  18/19 identical.
- **Redirect target confirmed**: our row 287 = `0x078209c2` (chain 2, offset 39); forcing
  bit 9 → offset 47; row 288 (`0x00590bc2`) already owns chain 2 / offset 47, so the
  last-writer-wins cover argument holds on our die.
- **⚠️ CORRECTION — your published burn list is wrong.** `0x0923` → addr 9193 requires
  burning record bits **[2,7,9,10,15]** → target `0x8FA7` (data bit already 1, cannot be
  cleared). Your stated [2,6,7,10,12,13] decodes to `0x3DE7` = addr 3961 = row 123 bit 25 —
  a different redirect into the hard-fuse region entirely. Under OTP monotonicity anyone
  burning your list gets row 123 and has burned the slot for nothing. Please re-derive and
  publish an erratum.

## 3. What the redirect actually buys — an open caveat

Row 287 is chain 2 / offset 39 with data `0x0782` (five set bits: 1,7,8,9,10). We find **no
Gen3-specific chain** anywhere in either dump, and no field map tying offset 39 to
`OPT_DISABLE_GEN3_SPEED`. Orphaning the block removes five set bits' worth of configuration,
not one flag. If you have shadow-log evidence from a burned card showing exactly which OPT
registers moved after the bit-9 divert, that would settle it — please share.

## 4. A second derived recipe: 64 → 96 GB via FF append (Hynix 8 GB cards)

OPT_FBP_DISABLE is an FF record (chain 1, offset 19) — appendable. On our card it carries
`0x852`: bits 6,11 are defective stacks (mirrored in hard rows 26/27), but **bits 1,4 are
disable-only and correspond to two full, healthy 16 Gb Hynix stacks (+32 GiB)**. Append
record **`0x420004C1`** (chain 1, offset 19, data `0x4200`) at any blank row above the
original → last-writer-wins yields `OPT_FBP_DISABLE = 0x840`, clearing exactly bits 1 and 4.
39 blank FF rows available on our die.

Open gates we will not burn past: (a) no documented successful FF-pool analog write (your
PG199 FF row failed analog while FPF rows burned — same observation here, no attempts);
(b) CFG1/LMR geometry values for 20-FBPA / 16 Gb dies unknown; (c) FB-Falcon retrain
behavior for revived channels unconfirmed.

## 5. The trade we're proposing

We can offer: the full layout decode + per-instruction maps for both SKUs, all anchors, the
burn-list corrections, our raw 512-row dumps, and the 96 GB append derivation.

We need, in exchange, any one of:
1. **Primary burn evidence**: source + raw logs of any successful RIR or FF-pool write on a
   GA100 (register sequence, payload semantics — delta vs full word, pulse/timeout handling,
   readback before/after cold cycle).
2. **A same-macro sacrificial card** result: slot state before/after, what the analog burn
   actually did.
3. **FBP-revival geometry**: CFG1/LMR for 20-FBPA configs if anyone holds them, or shadow
   logs from a card where FBP-disable bits were cleared.

Findings attributed by date; no names. Happy to file this as wiki pages if that's the
preferred venue.
