# Cooling the card

## What this page covers

Every cooling solution that has been measured on a CMP 170HX (or on an A100, which shares
the PCB and the heatsink), what wattage each one actually removes, which marketed claims are
false, and a recommendation table by target power limit. The thermal limits, sensor list and
temperature reference tables live on [Thermals](../hardware/thermals.md).

The headline, because it is the most expensive mistake people make:

> [!CAUTION]
> **The 3.24 W snail fan sold as an \"A100 cooler\" does not do 300 W**
>
> The widely sold 3D-printed "A100 cooling" adapter bundled with a **3.24 W, 12 V snail
> fan** is advertised as blowing off 300 W. First-hand measurement by a purchaser puts it
> at **150-180 W maximum**, at maximum duty cycle, powered directly from the PSU rather
> than throttled by a header. That is well short of the card's 250 W nominal envelope and
> nowhere near the 300 W OC-VBIOS envelope. Nobody contradicted the measurement. A second
> participant separately queried whether a radial fan can do this job at 3 W at all,
> noting that most radial fans of the required class draw at least 3 A (about 30 W).

The card is **fully passive**, reports `Fan Speed : N/A`, and will not protect itself with a
fan it does not have. Sizing rule: pick your cooler for the power limit you intend to run,
using the measured column below and not the marketing column.

---

## Five rules that fall out of the measurements

1. **Pull, do not push.** Sleeve-style shrouds that push air through the card produce
   "considerable blow back". Pulling air through the fin stack works much better in every
   first-hand account.
2. **Screw the duct on.** Friction-fit printed ducts "simply fall off" in use, in both
   single- and dual-GPU versions. The card has screw holes; use them.
3. **Wattage is the proxy, not RPM.** The useful spec is how many watts a solution has been
   shown to remove while holding 70 C core / 75 C memory, not the fan's rated speed.
4. **A single 40 mm fan fails, whatever its RPM.** One Arctic S4028-15K gives a 90 C
   hotspot; two give 70 C GPU / 76 C hotspot. An 8000 RPM 40 mm fan was abandoned by its own
   prospective buyer once the 80 C throttle onset was discovered ("guess 8k rpm won't do").
5. **Cool the VRM too.** Any solution that only covers the die, including a deshroud plus a
   pressure-mounted tower cooler, still needs a separate 80-120 mm fan over the power stages.

---

## Blowers, measured

| Solution | Measured result | Conditions | Confidence |
|---|---|---|---|
| Printed "A100 cooling" adapter + **3.24 W 12 V snail fan** | **150-180 W maximum** (advertised 300 W) | max duty, direct PSU feed | high |
| **EFB0251S3** blower, also rated **3.24 W** | **saturates at 73 C** at 200 W | `-pl 200`, sustained 100% load | medium |
| **San Ace B97, 1.85 A** (BFB1012VH) + USB fan controller, curve capped at 66% | **below 65 C at 250 W sustained** | curve driven from hotspot; it hunts, die temp is the better input | medium |
| **San Ace B97, 1.5 A** + pot-driven PWM controller (PCIe or SATA powered) | **claimed 350 W, unmeasured on a 170HX** | unusable at full blast without a controller | low |
| **9733S 12 V 2-wire blower** (97 x 97 x 33 mm) + Level1Techs printed shroud | idle **64 C to 36 C**; load **80+ C to 40-50 C** | **on an A100**, same PCB and cooler, LLM workloads | medium |
| **AC Infinity S2-class blower, no shroud** | **150 W OK, throttles at ~180 W** | **on an A100** (85 C limit), large air-leak paths | medium |
| **AC Infinity S2-class blower, no shroud** | **200 W** stress test passed | **on an MI100** | medium |
| Cheap 40/60 mm blower taped on with **aluminium tape** | no temperature posted | reported surprisingly sturdy, and loud | low |

> [!NOTE]
> **Open problem: two nominally identical 3.24 W blowers, two different outcomes**
>
> The printed adapter with a 3.24 W snail fan tops out at 150-180 W. An EFB0251S3, also
> stated at 3.24 W, holds a card at 73 C at a 200 W limit under sustained full load. Same
> nominal fan power, materially different result. Duct sealing and the static-pressure
> path are the obvious explanation, and the tester who measured the unshrouded case
> reported "There are huge spots for leaks", but **nobody has run the two side by side on
> the same card at the same power limit**. Until someone does, treat the 3.24 W class as
> a 150-200 W class and assume duct quality is doing most of the work.

Treat **250 W as the proven figure for the B97 and 350 W as an unverified claim.** The 350 W
number is confidently asserted and supported by the same blower cooling a 350 W FPGA for a
year, but no measured 350 W log on a 170HX exists.

---

## Axial fans and shrouds, measured

| Solution | Measured result | Conditions | Confidence |
|---|---|---|---|
| **1 x Arctic P12 Pro PST 120 mm** through a printed shroud, feeding **2 cards** | **32 C idle** at ~34 W each | 1000 RPM (fan capable of 3000), electrical tape added around the shroud to raise static pressure | high |
| **2 x Arctic P12 Pro PST CO 120 mm** in printed ducts with internal splitter vanes, feeding **4 cards** | **under 65 C** | `-pl 160` per card, "not much noise" | medium |
| **1 x 120 mm @ 3000 RPM**, ducted | **73 C** peak | stock clocks, "a bit too loud" | high |
| **Cooling unstated** (one owner, one setup) | **~70 C ±2** default clocks; **75-77 C** overclocked | up to 300 W. The fan was never named in the source thread | high |
| **Arctic 12038-4K 120 mm** | **~85 C**, "cannot cool it any better without water" | memory-overclock `gpu_burn`; HBM errors appeared within the first couple of minutes, cause unresolved | medium |
| **1 x 140 mm Noctua NF-A14 industrialPPC** + repurposed P100 shroud, 3 cards | **~38 C idle, under 80 C** loaded | wattage-driven custom fan curve, quiet at idle | high |
| **2 x stacked Noctua 140 mm** | **~80 C down to ~70 C** | adding the second fan | high |
| **80 mm 10k-RPM server fan @ 3500 RPM** | **under 70 C core / 75 C memory** at 200 W | same setup handles 300 W without difficulty | medium |
| **80 mm server fan @ 7500 RPM** | **70 C core / 75 C memory** at full 300 W | with an overclock | high |
| **80 mm server fan @ 35% speed** | within 70/75 C | `-pl 175` | high |
| **2 x Arctic S4028-15K 40 mm** on a custom 2-slot bracket | **never above 70 C GPU / 76 C hotspot** | curve set to 100% at 70 C, fans hit 15000 RPM | high |
| **1 x Arctic S4028-15K 40 mm** | **90 C hotspot, rejected** | same bracket | high |
| **Two-fan printed shroud shipped with some cards** | never above 60 C | under half fan speed, card not yet unlocked, workload unspecified | medium |
| **Stock passive heatsink in a datacenter chassis** (Gigabyte G292-Z20, 80 mm fans) | **peak 60 C at 254 W** | 8-card rental; "louder than a jet engine" | medium |
| **Deshroud + 2-3 x 120 mm Noctua** directly onto the opened fin stacks | untested | proposed alternative to ducts | low |
| **Deshroud + pressure-mounted tower cooler** | untested | "gravity or zip ties works", no clamping pressure needed; VRM still needs its own 80-120 mm fan | low |

The 80 mm high-RPM server-fan rows are the strongest air results in the corpus: they meet
the 70 C / 75 C design target at the full 300 W envelope, and at 200 W they do it at only
3500 RPM.

---

## Shrouds and ducts

| Design | Verdict |
|---|---|
| **Level1Techs A-series blower adapter** (`l1 a100 blower.stl`, 52.3 KB) | Best-evidenced printed part. Support-free print that bolts to the existing screw holes at the far end of the card, with an angled low section beside the power connector for finger access. Produced the 64 C to 36 C idle result on an A100 with a 9733S blower. Caveat: the card becomes noticeably longer. Filament, print settings, airflow direction and durability were asked in-thread and never answered. |
| **Thingiverse 5532715** (single and dual 120 mm) | Usable; **use the "fixed" variant**. The dual version is too long for some rigs. |
| **`CMP_170HX_Fan_Shroud_Fixed.stl` / `CMP_170HX_Dual_Shroud_Fixed.stl`** | Friction-fit, falls off over time, walls thin enough to be weak even in PETG. Only STLs exist, so modification needs a re-model rather than a STEP edit. |
| **Cults3D `cmp-170hx-fanduct`** (single and dual GPU) | **Mechanical failure.** Both versions "simply fall off". |
| **LTT shroud** | **Rejected by users.** "The ltt shroud is awful", "Lots of back-pressure." |
| **Sleeve-style shrouds / push-through airflow** | **Rejected.** Considerable blowback; will not work in 2-slot spacing. Flush bolt-on brackets will. |
| **A5000/A6000 blower shrouds as a drop-in** | **Disputed, leaning against.** The proposal rested on shared Ampere workstation PCB dimensions; the rebuttal cited different screw holes and no fan connector. Neither side posted a photo of an attempted fit. |
| **A100 shrouds** | **Unresolved.** Asserted to fit ("Only difference is the text right?"), never photographed or measured on a 170HX. Note that shared-PCB reasoning has already failed once here, with the A5000/A6000 claim. |
| **Boring out the stock two-slot shroud for an integral fan** | Works for a single card or spaced-out cards; fails for adjacent installs because two-slot cards leave no intake gap. Never built. |

Practical gotchas reported repeatedly: several shrouds **block access to the card's EPS power
plug**, so test-fit with the cable in place; third-party print quality is unreliable (one
part described as 120 mm arrived as 140 mm); and use **aluminium tape, not duct tape**, if
you are taping a blower on.

> [!NOTE]
> **Open problem: print material**
>
> PLA is reported as fine in practice by a multi-card user ("pla has been fine for all
> mine"), on the argument that the duct sits in cool intake air away from the heat source.
> Others recommend PETG or ASA as safer near GPU heat. Mildly disputed, no failure
> reported on either side.

---

## Water cooling

Water is the only route that has produced sub-50 C load temperatures on this card.

| Block | Measured result | Notes |
|---|---|---|
| **Bykski N-TESLA-A100-X-V2** + 360 mm 3-fan radiator | **30 C idle at 30 W; 45 C after 30 min at 180 W** with fan and pump both at **minimum** speed | The only commercially available full-coverage block confirmed compatible. Also fits A100 40 GB PCIe, A30 24 GB and Tesla L40 (shared PCB). Uses standard G1/4 fittings. Lower temperatures are available at higher fan speed. |
| **Bykski N-TESLA-A100-40G-SR** + Bykski 360 mm radiator, three 120 mm PWM fans, D5 pump | **53 C core / 58 C memory at 250 W** (`gpu_burn`, radiator fans 100 %, pump 55 %, heat-soaked loop, 23 C room); **55 / 59.5 C** with the radiator fans at 50 % (about 1660 rpm) | One report. Fits, but needed hand-made fixes at two spots (see the first subsection below). Pump speed barely matters: 20 % to 100 % moved the result by 1 to 2 C. |
| **Budget generic full-cover block** | **48 C core / 58 C memory** after a 1-hour stress test | 8 GB card, in service about a year. |
| **EK-PRO GPU WB RTX A100** | no temperatures posted | Fits. Out of production. |
| **Chinese V100 SXM2 flat-plate block** | untested | Plain flat plate with no conforming surfaces; the MOSFET and inductor gaps would have to be filled by hand. |
| **Arctic Liquid Freezer II 420 AIO + copper adapter plate** | **FAILED** before a game finished loading | Photographed **pump-out**: cracked paste with bare copper across much of the die contact area. Attributed to poor mounting pressure or an incompatible adapter. |

> [!CAUTION]
> **Fitment traps that destroy cards or waste money**
>
> - **`N-TESLA-A100-80G-X-V2` does not fit.** The A100 80 GB is a different, incompatible
>   PCB. Only the 40 GB-family block is correct.
> - **The V2 (all-metal) revision is required.** The earlier non-V2 uses transparent
>   acrylic.
> - **SXM waterblocks do not fit PCIe cards at all.** At least a third of A100s sold were
>   SXM. There are also two different PCIe A100 block designs, because the A100 80 GB has
>   no integrated heat spreader while other variants do: "if that waterblock is made for
>   the variant without IHS it wont work properly."
> - The correct V2 block ships with **no manual and a hex wrench of the wrong size**. Have
>   a metric hex set ready.

> [!CAUTION]
> **The single most damaging waterblock installation mistake**
>
> **Cover every unpopulated IC footprint within reach of the block's contact pillars with
> thermal pad before lowering the block.** Because the 170HX is a depopulated A100 board
> it has far more bare footprints than a real A100, and the block's metal pillars can
> short across exposed copper pads and permanently kill the card. Pad the DrMOS MOSFETs
> left and right of the ASIC, the areas bottom-left and bottom-right of the ASIC, the PMIC
> to the right of the die between an inductor and a capacitor, and the two PMICs to the
> left of the die below the 3.3 µH inductor. Thermal paste (pea-sized) goes only on the
> GPU/HBM copper spreader. The author of this procedure lost a card about an hour after
> installation and this is their leading suspicion, competing with a pre-existing
> mining-wear fault; it was never definitively isolated.

Full teardown order, washer and bracket reuse, and the 15-minute pressurised leak test are
on [Physical mods](physical-mods.md).

> [!NOTE]
> **Open problem: HBM temperature on water**
>
> The one sweep that tried to lower HBM temperature on air concluded "getting lower hbm
> temps on air seems impossible". A request for equivalent waterblock memory-temperature
> data went unanswered, so nobody knows whether the HBM floor is a cooling limit or
> intrinsic to the package. The budget-block owner who measured 58 C memory already has
> the hardware to answer it.
>
> One tester's loop (see the last subsection below) reports HBM never below 44 to 46 °C at idle and 51 to 52 °C under load whatever the fan and pump speed, and 58 °C at 250 W once heat-soaked, close to the 58 °C reported above for the budget block. That suggests the floor is set by the package and its contact rather than by the loop, but it is two loops and has not been reproduced; the absolute accuracy of the HBM sensor is unknown.

### Fitting the Bykski N-TESLA-A100-40G-SR (one build)

One tester fitted this 40 GB-family PCIe block, which comes with a manual, to an 8 GB 170HX (unlocked to 64 GB), and reported that it fits almost perfectly, but that **two components were not reached by the contact surface as delivered** and that the block has more raised bosses than the manual covers. A test fit showed where (thermal pads on the chips, block lowered and removed again; a pad that stuck to the block shows contact, a pad that stayed on its chip with no mark on the block shows a gap):

- **Spot 1, a pocket above a component.** A component between two inductors on the PCIe-connector side of the GPU sits under a large recess in the block, so a pad alone leaves a gap. Fix used: a 15 x 7 mm rectangular aluminium spacer cut from an old heatsink, with a thermal pad on each side (pad, spacer, pad). The spacer was 5.5 mm tall, with the kit's 1.7 mm pads above and below it. The tester found the height by 3D-printing 15 x 7 mm blanks in 0.5 mm steps and test-fitting each.
- **Spot 2, a flat area with no boss.** A small IC near the power-connector end of the board faces a flat surface of the block, so the gap is simply the missing height. Fix used: two of the kit's 1.7 mm pads glued together.
- **Extra bosses.** The block has further raised pads, beside the capacitors on the left and the right of the GPU, that the manual does not say to pad. The tester matched them to components and fitted pads there as well.
- **Capacitor insulation.** After reading about components touching the block, the tester covered two areas of the block, where the capacitors seemed able to touch it, with Kapton (polyimide) tape. A precaution by eye, not measured.
- **Soft pads.** The kit's pads (1.7 mm) were very soft and looked like the card's factory pads (both cards opened had intact factory seals). A stiffer set bought separately was not used: a soft pad can be pressed down with light force, so a gap that is wrong by a fraction of a millimetre still gives contact without bending the card or straining the PCB.

> [!CAUTION]
> **The aluminium spacer conducts.** It must be insulated on both faces by pads that are larger than the spacer and do not conduct electricity, with no overhang or burr touching anything else. Check for shorts before powering up. Stacked pads also conduct heat worse than a single pad, and the chips involved have no temperature readout (unknown), so there is no number for them; in this build the card ran `gpu_burn` at 250 W without errors or throttling.

### Pump and radiator-fan duty on a water loop (one measured loop)

One tester measured pump duty and radiator-fan duty against temperature on a 250 W-limited unlocked 8 GB card (`10de:20c2`, 64 GB) with `gpu_burn` FP32 on a custom loop: Bykski N-TESLA-A100-40G-SR block, Bykski 360 mm radiator, Bykski D5 pump with a 150 mm reservoir, three 120 mm ARCTIC P12 Pro PWM radiator fans, 23 C room. The pump sits on a motherboard `AIO_PUMP` header and each radiator fan on its own chassis header (one common duty), all driven through the Nuvoton NCT6798 hwmon driver. Each step ran 5 to 6 minutes after a warm-up; the plateau is the median of the last 2 minutes.

| Radiator fan duty | Radiator rpm | Core / HBM °C | Evidence | Conditions |
|---|---|---|---|---|
| 100 % | 2900 | 53 / 58 | one report | pump 55 %, 249 W, heat-soaked loop |
| 70 % | 2200 | 54 / 59 | one report | same |
| 50 % | 1660 | 55 / 59.5 | one report | same |
| 35 % | 1270 | 56 / 59 | one report | loop not fully heat-soaked, perhaps 1 to 2 °C optimistic |
| 25 % | 1010 | 59 / 62 | one report | same |
| 15 % | 740 | 64 / 66, still rising 0.7 °C/min | one report | not settled after 8 minutes |

| Pump duty | Pump rpm | Core / HBM °C | Evidence | Conditions |
|---|---|---|---|---|
| 100 % | 4070 | 56 / 61 | one report | radiator fans 50 % (about 1660 rpm), about 245 W |
| 70 % | 3750 | 55 / 59 | one report | same |
| 55 % | 3480 | 55 / 59.5 | one report | same |
| 40 % | 3125 | 55 / 59 | one report | same |
| 30 % | 2820 | 55 / 60 | one report | same |
| 20 % | 2480 | 56 / 61 | one report | same |

What the numbers say, for this loop:

- Radiator fans: going from 50 % (about 1660 rpm) to 100 % (about 2900 rpm) lowered the core by about 2 °C and HBM by about 1.5 °C. Below 35 % the temperature rises quickly.
- Pump: across 20 % to 100 % (2480 to 4070 rpm) the spread was 1 °C on the core and 2 °C on HBM. Pump speed above its minimum bought nothing measurable.
- Two fan curves built from this were validated with 10-minute 250 W runs: radiator and pump at 30 % until the core reached about 60 °C gave 59 / 62 °C at about 1100 rpm radiator and 2870 rpm pump; radiator up to 50 % and pump 40 to 50 % gave 55 / 60 °C. No compute errors in either run.
- Noise (one listener, quiet room): the three fans were indistinguishable from off up to 20 % duty, almost inaudible at 30 % (about 1100 rpm), quiet at 40 %, audible from 50 %, and uncomfortable at 70 % (about 2200 rpm). The pump was indistinguishable between 20 % and 30 % (about 2500 to 2800 rpm), audible as a thin whine up close at 40 %, and audible across the room at 55 %.

HBM temperature in this loop: never below 44 to 46 °C at idle, whatever the fan and pump speed, and never below 51 °C under load (about 157 W, loop not yet heat-soaked) or 52 °C at 250 W; with a heat-soaked loop and the radiator fans at 100 % it was 58 °C. HBM sat about 15 °C above the core at idle and 5 to 12 °C above it under load in every run.

---

## Fan control and noise

There is no usable fan control on the card itself. Everything is external.

- **No confirmed on-card fan header.** An unpopulated pad that looks like a fan location was
  spotted on the card. Asked whether it is a 4-pin, one participant answered that it carries
  12 V and ground only; a skeptic in the same thread said it "doesn't look like a fan con".
  Nothing was ever probed, and no photo-confirmed pinout and no working fan install was ever
  posted. Even if it is a header, the reported pins are power only, which would mean a fixed
  full-speed fan with no tachometer.
- **Control the fan from the host.** A USB or pot-driven PWM controller is what testers
  actually used. Drive the curve from **die temperature**, not hotspot, because hotspot-driven
  curves hunt.
- **A wattage-driven curve** was reported as the quietest approach on a 3-card rig with
  140 mm industrial fans.

Noise, in the order testers described it:

| Setup | Noise |
|---|---|
| 1 x P12 Pro 120 mm at 1000 RPM feeding 2 cards | quiet |
| 2 x 120 mm feeding 4 cards at `-pl 160` | "not much noise" |
| 140 mm industrialPPC on a wattage-driven curve | quiet at idle |
| 1 x 120 mm at 3000 RPM ducted | "a bit too loud" |
| San Ace B97 at full blast without a controller | unusable |
| 2 x S4028-15K 40 mm at 15000 RPM | loud by design |
| Stock passive heatsink in a datacenter chassis | "louder than a jet engine" |

One more practical annoyance: a fixed-speed fan on a card with no fan control will "kick in
at 100% speed during a system reboot", which is startling the first time.

---

## Recommendation by target wattage

Pick the row for the power limit you will actually run, then verify with an **integer or
memory** benchmark, never a conventional FP32 burn-in (see
[Power and PSU](power-and-psu.md) for why FP32 only draws ~60 W on this card).

| Target | Recommended | Evidence | Avoid |
|---|---|---|---|
| **Idle / light** (~30-45 W) | Any 120 mm fan through a screw-on duct at low RPM | 1 x P12 Pro at 1000 RPM held **32 C** across 2 cards | Nothing at all: passive with no airflow leads to leakage runaway |
| **150 W** (`-pl 150`) | 3.24 W-class blower with a well-sealed printed adapter, or a single ducted 120 mm | Snail-fan adapter proven to **150-180 W**; AC Infinity S2 class OK at 150 W on an A100 | Any unshrouded blower; leaks destroy the static-pressure path |
| **160-200 W** (`-pl 160` to `-pl 200`) | 1 x 120 mm per 2 cards in a vaned duct, or an 80 mm 10k server fan at ~3500 RPM | 2 x 120 mm held **4 cards under 65 C** at `-pl 160`; 80 mm at 3500 RPM held **under 70/75 C** at `-pl 200`; EFB0251S3 saturated at **73 C** at 200 W | Single 40 mm fan (90 C hotspot) |
| **250 W** (stock cap) | San Ace B97 1.85 A + PWM controller, **or** 2 x 40 mm S4028-15K on a 2-slot bracket | B97 held **below 65 C at 250 W sustained**; 2 x S4028-15K held **70 C / 76 C hotspot**; one owner sat at **~70 C ±2** at default clocks with the cooling never described | The 3.24 W snail-fan adapter (it tops out 70-100 W short) |
| **300 W** (OC VBIOS only) | 80 mm server fan at 7500 RPM, or water | 80 mm at 7500 RPM met **70 C / 75 C** at full 300 W; one 30 min `gpu_burn` at a 300 W limit held **75-77 C** with the cooling unstated (a single report, not reproduced) | Thin 120 mm fans, doubted for 300 W and especially for two cards. One Arctic 12038-4K owner sat at ~85 C and could go no further on air |
| **Quietest at any wattage** | 360 mm loop with a compatible full-cover block | **45 C at 180 W with fan and pump at minimum**; 48 C core / 58 C memory on a budget block at load | AIO plus adapter plate: the one documented attempt pumped out and failed |
| **Multi-card, dense** | 80 mm high-RPM server fans per card, or ducted 120 mm with splitter vanes | 4 cards under 65 C on 2 x 120 mm at `-pl 160`; 8-card datacenter chassis peaked at 60 C at 254 W | Two single shrouds side by side: they will not fit two adjacent cards |

Before buying anything, note that raising the power limit buys very little on this card
(+2.8% BF16 from 250 W to 300 W, with temperatures already below 65 C), so cooling for
`-pl 160` to `-pl 200` is a defensible choice on efficiency grounds alone. The measured
efficiency peak of 1390 GFLOP/W was recorded at a 1400 MHz ceiling with a +350 MHz offset drawing
about 134 W, but that cell was never validated with a memory sweep and sits between 1400/+325
(silent corruption) and 1400/+375 (fault), so it is not an operating point. Neither is the
1376 GFLOP/W cell at 1350 MHz / +300, which is also a single completed run that no pattern sweep
ever gated. The highest sweep-validated efficiency point is the shipped `eff` profile,
1366 GFLOP/W at +250 / 1350 MHz. See [Tuning](tuning.md).

---

## Claims shown to be false

| Claim | Status |
|---|---|
| The 3.24 W snail-fan "A100 cooling" adapter handles 300 W | **False.** Measured 150-180 W maximum. |
| Friction-fit printed ducts are adequate | **False.** Both single and dual versions fall off. |
| A single 40 mm fan is enough at high RPM | **False.** 90 C hotspot on an S4028-15K; the 8000 RPM proposal was abandoned by its proposer. |
| Sleeve shrouds pushing air through the card work | **False.** Considerable blowback; pulling works much better. |
| A5000/A6000 blower shrouds drop in | **Contradicted** (different screw holes, no fan connector), though not photographed either way. |
| The LTT shroud is a good option | **Rejected by users** for back-pressure. |
| 90 C is a fine operating temperature | **False.** Throttle onset observed at 80 C; design target is 70 C core / 75-76 C memory. |
| Better cooling plus 300 W unlocks significant performance | **False.** Measured +2.8% BF16, with core and memory both under 65 C. |
| An AIO cold plate on the IHS via a copper adapter works | **Failed** in the one documented attempt: photographed pump-out. |
| SXM waterblocks fit with straps and a thick pad | **False.** Physically incompatible with PCIe cards. |

---

## See also

- [Thermals](../hardware/thermals.md): limits, sensors, and the full measured-temperature
  reference.
- [Power and PSU](power-and-psu.md): power limiting, and why FP32 burn-ins do not load this
  card.
- [Power delivery](../hardware/power-delivery.md): where the VRM sits and why it needs air.
- [Physical mods](physical-mods.md): teardown sequence, waterblock install, capacitor mod.
- [Tuning](tuning.md): the clock and power sweep that produced the efficiency numbers.
