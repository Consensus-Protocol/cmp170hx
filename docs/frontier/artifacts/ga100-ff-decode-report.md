# GA100 offline FF decode and shadow anchors

## Scope and evidence
Read-only analysis of `/home/f0ol/fuse_dump_results.json`; no MMIO access, driver changes, fuse commands, or files in cmpunlocker modified. Reproduce with `python3 /home/f0ol/ga100-ff-analysis/analyze.py`. Complete decoded list: `instructions.csv` / `instructions.json`; full aligned two-card table: `decoded-map.md`; candidate scores: `scores.json`; field search results: `field-matches.json`.

## 1. Chosen layout (empirically strong, formal width not fully identifiable)
Working decode: `chain = word & 0x3f; offset = (word >> 6) & 0x3ff; data = word >> 16`. Thus chain [5:0], offset [15:6], data [31:16]. Offset is a 16-bit block index, not an OPT register index. Fields need not be aligned within a data word.

All 145 nonzero records have bit 5 clear, so a five-bit chain plus a reserved/flag bit at bit 5 is observationally indistinguishable. Upper offset width is likewise not proven: Samsung uses offsets 291/292, demonstrating why the original seven-bit extraction is incomplete if these bits are offset bits. All 16 bits of the low half are assigned by the proposed layout, but other interpretations of unused bits are not excluded. No independently established separate 'type' field exists in these dumps.

Hynix: 71 nonzero rows (222–292), chains 1:16, 2:55. Samsung: 74 nonzero rows (222–295), chains 1:15, 2:56, 6:2, 23:1. No duplicate chain/offset keys on either card. 69 shared keys, 76-key union; 69/71 Hynix keys and 69/74 Samsung keys align despite reordered physical rows. Known floorsweep masks and device IDs decode correctly.

The external histogram totals **103**, including **29** unknown-chain instructions, not 71. It cannot describe this Hynix dump under one instruction per nonzero row. Its chain-1 count matches; chain 2 differs by two; its other chains are absent. Do not bend the decoder to reproduce another card's or an erroneous histogram.

### Scoring
Transparent heuristic, not a confidence probability:
- H = 1 - sum_c |observed_count[c] - external_count[c]| / (71 + 103).
- J = intersection / union of cross-card (chain,offset) keys.
- P = (fraction of Hynix offsets <=127 + fraction of unique Hynix keys)/2. Compactness prior only; not independent proof of width.
- Score = 100*(0.4 H + 0.4 J + 0.2 P).

| Layout | H | J | P | Score |
|---|---:|---:|---:|---:|
| Low 6-bit chain, offset [15:6], high-half data | .79310 | .90789 | 1 | 88.040 |
| Same, truncated 7-bit offset [12:6] | .79310 | .90789 | 1 | 88.040 |
| Low 5-bit chain, offset [15:5], high-half data | .79310 | .90789 | .85211 | 85.082 |
| Low 7-bit chain, offset [15:7], high-half data | .37931 | .90789 | 1 | 71.488 |
| Best mid-low chain / high-half data | .39080 | .90789 | .60563 | 64.061 |
| Best low-half data alternative tested | .31034 | .22609 | .65493 | 34.556 |

The tied truncated-offset score is expected: Hynix lacks offsets above 86 and Samsung's high-offset records are unmatched either way. Prefer the complete 6/10/16 partition as the working model, not as a mathematically unique deduction. Existing offset39→47 evidence independently favors a field beginning at bit6.

## 2. Decoded map and identity/floorsweep fields
See `decoded-map.md` for every row/data value on both cards; offsets decimal, data hex.

| Chain | Hynix offsets | Samsung offsets |
|---|---|---|
| 1 | 4–9,11–12,14–16,19–21,23,26 | Same except no 21 |
| 2 | 10–23,25,37–39,46–47,49,51–75,77–81,83–86 | Same except no 25; add 30,35 |
| 6 | None | 291,292 |
| 23 | None | 66 |

No whole data halfword equals 0x23, 0x852, 0x24, or 0x840. Correct matches are bitfields:

| Shadow / identity | Physical source | Extraction | Hynix | Samsung |
|---|---|---|---|---|
| OPT_GPC_DISABLE (0x820350) | chain1/off8: rows274/279 | (data>>3)&0xff | 0x23 | 0x23 |
| OPT_FBP_DISABLE (0x820364) | chain1/off19: rows280/248 | (data>>3)&0xfff | 0x852 | 0x024 |
| OPT_FBP_DEFECTIVE (0x8205cc) | hard rows26/27 | (row>>16)&0xfff | 0x840 | 0x024 |
| DEVIDA candidate (0x8204d8) | chain2/off37: rows285/288 | ((D37>>1) | ((D38&1)<<15)) | 0x20c2 | 0x2082 |
| DEVIDB candidate (0x82056c) | chain2/off38: rows286/289 | ((D38>>1) | ((D39&1)<<15)) | 0x20c2 | 0x20c2 |

FBP disable raw data is 0x4290/0x0120; GPC raw data is 0xf91e/0xf918. FBP mask 0x852 means bits1,4,6,11; 0x840 means bits6,11. These are strong empirical row-to-shadow anchors, with differing multibit FBP values reproduced on both cards. Exact numerical matches are confirmed from supplied observations; causality/wiring is not proven by interventions or NVIDIA fuse schema. Hard rows26/27, not merely rows28/29, directly contain the defective mask. DEVIDB=0x20c2 on Samsung independently agrees with the archived JRex comparison table for other 10GB cards, but our device-ID shadow itself was not present in the saved gpcprobe summary.

## 3. Gen3 story
**Structurally consistent; functional clearing and safety unverifiable with this data.** Hynix row287 decodes chain2/off39/data0x0782; setting row bit9 changes only offset39→47. Row288 is chain2/off47/data0x0059. Samsung has the same pair at rows290/291.

A pure last-writer-wins simulation changes only key (2,39), from present with data0x0782 to absent; (2,47) remains0x0059. This confirms the address-collision argument conditional on last-writer-wins, not the hardware semantics, absent-block reset value, or Gen3 bit identity. Data0x0782 has five set bits: 1,7,8,9,10. No evidence assigns them to Gen1/2/3 fields. There is no separate identifiable 'gen-speed chain': this is the mixed identity/configuration chain2, adjacent to DEVID fields. Losing a whole block can affect multiple settings. The supplied dumps contain no Gen3=0 control card and no duplicate FF target that could validate overwrite precedence.

**Address discrepancy:** the actual local `/home/f0ol/gpcprobe/gpc_probe.py` uses OPT_GEN23=0x82057c and OPT_DISABLE_GEN3_SPEED=0x820580 (lines118–119), independently matching the archived JRex table. Narrative notes instead assert 0x820250. That discrepancy must be resolved against an authoritative GA100 register definition before interpreting a new read. This analysis does not silently equate the two addresses.

## 4. Rows16–19 and RIR meaning
Rows16/17 are identical 0x60711465 on both cards; they are paired hard-fuse words, not FF instructions. Calling them a header/config remains a hypothesis without field-map evidence.

RIR substitution is established at the raw-read level: Hynix row18 bit8=1; row19 bit8=0; Samsung both0. The effect of that bit on OPT registers is **unknown**. The supplied known GPC, Gen23, Gen3, ECC and override values do not show a distinguishing Boolean value tracking it. It therefore cannot be assigned directly to any of those single-source one-bit shadows from this comparison. Redundant-copy combining, ignored/reserved bits, unmeasured configuration, or a repair of one copy could all hide or change its downstream effect. A raw-read change alone proves neither an effective functional change nor a harmless no-op. Factory intent is not recoverable here, and redirecting away from the original bit cannot be declared safe.

Additional input correction: source JSON rows8/9 differ (Hynix0xc0480e89 vs Samsung0x14380e89); only rows0–7, not rows0–9, are identical.

## 5. Highest-information read-only follow-ups
1. Obtain the actual GA100 FF/hard-fuse schema or original external decoder plus its matching raw dump and exact before/after Gen3 shadow logs. Analyze offline. This can identify chain2/off39 bits, row18/bit8 and field widths, and settle the 103-vs71 mismatch without a burn.
2. Acquire an additional matched main-array+OPT-shadow dataset from a Gen3-enabled GA100 control card; prioritize one with identical low hard-fuse configuration and differing chain2/off39. Differentially map all five bits in0x0782 rather than assuming one is the entire Gen3 gate. Prefer existing captures; any acquisition needs separate authorization.
3. Capture a comprehensive read-only OPT shadow comparison for these two cards (including documented 0x820580 and the separately labelled disputed0x820250, DEVIDA/B, hard-fuse-related security/configuration shadows), then compare additional same-SKU dies with/without the row18 substitution. Independent variation is essential; two different-SKU cards alone leave factory repair vs SKU behavior confounded. No fuse programming, software overrides, controller writes or reset experiments are part of this deliverable.

## Verification / limits
Decoder assertions passed, all145 nonzero instructions saved, zero duplicate keys, and scoring reproduced. `/tmp/gpcprobe-out.txt` is absent, so saved summary readings rather than original per-register output were used. No authoritative fuse-bit schema was found in the supplied material. FF layout widths and row18 functionality remain explicitly unresolved; do not promote the prior 'Gen3 story verifies / safety holds' claim to a hardware conclusion.
