# OwlSat Radiation Survival — Findings & Reference Summary
###### Courtesy of Opus 4.8

*Synthesis of the radiation literature in `radpapers/`, written to answer the questions in
[`radiation-papers.md`](radiation-papers.md). Searchable by component, effect, and question.
Last updated 2026-06-22.*

> **How to read this doc.** Part 1 is a TL;DR with direct answers to every question in the brief.
> Part 2 is the supporting detail (environment, per-component, mitigation, rules of thumb). Part 3
> summarizes each source paper so you can grep for a number and find where it came from. Part 4 is
> the "read these next" shortlist.

---

## Part 1 — Direct answers to the brief

### Q: RP2350B + AS3016204 STT-MRAM — what failures should we worry about?

**RP2350B (Cortex-M33, 40nm-class TSMC bulk CMOS):**
- **SEU/SEFI** in SRAM, registers, and control logic is the dominant concern — these are *recoverable*
  (scrub/reset) and at LEO rates are infrequent but not negligible. This is what you design software around.
- **SEL** is the one that can *destroy* the part. A modern sub-90nm bulk CMOS device is **not** inherently
  latch-up immune (TI handbook §5.2: "Starting at around the 90-nm node and smaller, most CMOS processes
  are likely to experience SEL on standard CMOS structures without guard rings"). RP2350 has *no published
  SEL-free guarantee*, so you must treat SEL as possible and design the power architecture to detect-and-cycle.
  This validates your ADM1176/PMOS-switch plan (see SEL section).
- **TID** is a slow background concern, not your limiting factor in LEO (see dose budget). Deep-submicron
  bulk CMOS routinely survives 300 krad → Mrad of TID; the RP2350 will almost certainly outlast your mission
  dose. The risk inside an MCU from TID is increased static leakage (higher idle current) long before functional failure.

**AS3016204 STT-MRAM (Avalanche, 40nm):**
- The **magnetic storage (the MTJ bits) is intrinsically radiation-tolerant** — heavy ions and TID do not
  flip magnetic state (confirmed for MRAM generally in the Freescale MR2A16A report and the Avalanche
  Goddard data). **The vulnerability is the CMOS periphery** (sense amps, charge pumps, address/control
  logic) — same SEU/SEFI/SEL exposure as any CMOS chip.
- For *your specific part* the Goddard compendium (Topper 2020) is good news — see the dedicated MRAM
  question below.

**Bottom line:** plan for SEU (software scrub/EDAC/watchdog) and SEL (current-limit + power-cycle). TID is
not your binding constraint in LEO for either part.

### Q: Which radiation measurements matter most?

In rough priority for a LEO COTS CubeSat:
1. **SEL threshold LET (LET_th) and latch-up cross-section** — because SEL is the destructive,
   non-recoverable-by-reset failure. A part with **LET_th ≥ ~60–80 MeV·cm²/mg is effectively SEL-immune
   to the trapped/most-GCR spectrum**; LET_th below ~15 is a real concern.
2. **SEL holding current / measured latch current** — sizes your current limiter (see "common SEL currents").
3. **SEU cross-section vs LET** (σ_sat in cm²/bit and LET_th) — lets you estimate bit-error rate.
4. **TID to functional failure** (krad(Si)), *and the bias/dose-rate conditions of the test* — TID numbers
   are nearly meaningless without stating biased vs unbiased and dose rate (ELDRS).
5. For power MOSFETs only: **SEB/SEGR safe-operating-area (derated V_DS) vs LET**.

LET is the universal x-axis (MeV·cm²/mg). Cross-section (cm²/device or cm²/bit) × particle flux = event rate.

### Q: Tolerance from flight-heritage / beam-tested parts (PyCubed, RP2350)?

- **Flight heritage tells you it survived *a* mission, not a quantified LET_th/TID number.** It is strong
  evidence a part works in LEO but is not a substitute for a beam test or a vendor SEE report. Treat it as
  "de-risked, dose-limited unknown."
- **Beam-tested COTS** (like RP2350 via Holliday, and the parts in the JPL/Guertin and Goddard compendia)
  give you the actual LET_th / cross-section / TID-fail numbers — vastly more useful. Where a beam report
  exists, use *its* numbers, not heritage.
- Typical COTS commercial silicon (the population PyCubed parts are drawn from): functional to **~2–10
  krad(Si)** at minimum, often far more for deep-submicron; SEU LET_th frequently low (≈1–5). Rad-*tolerant*
  parts: 20–50 krad. Rad-*hard*: >200 krad (Llis-824).

### Q: Tolerance from AEC-Q100/101/104/200 parts?

**AEC-Q qualification says *nothing* about radiation.** This is the single most repeated finding across
Sampson (NASA Automotive), Airbus (4.3 ADS Newspace), and EPCI:
- AEC-Q100/101/200 cover temperature grades, life, and electrical stress — **there is no radiation
  (TID/SEE) requirement and no lot-to-lot radiation control.** A tracecode/datecode does **not** let you
  infer radiation behavior; radiation must be managed per **diffusion lot / wafer fab**.
- AEC-Q also has **no inherent screening requirement** (Sampson) — "automotive grade" ≠ "screened part."
- *However*: automotive parts are built to survive a harsh, high-reliability environment, and Airbus/EPCI
  found that under proper derating their field failure rate is "not significantly different" from
  space-qualified parts for many categories. AEC-Q is a reasonable *reliability* starting point — you still
  must add your own radiation characterization or rely on per-part beam data.
- Practical Airbus number: ~77% of AEC-Q parts were usable at system level after screening (≈80% of
  passives, only 25–40% of actives, and **no** MOSFETs/RF without extra work).

**Use AEC-Q to bound thermal/reliability risk. Do not let it stand in for a radiation answer.**

### Q: Should we reconsider the Avalanche MRAM decision (vs Everspin)?

**No — your data actually supports keeping the Avalanche AS3016204, with one caveat to close.**

The Goddard compendium (Topper 2020, Table III) tested **two** Avalanche 40nm MRAMs and they behaved
differently — this is the source of your confusion:

| Part | SEE result | TID result |
|---|---|---|
| **ASV016204** (your decision basis) | **No SEEs** (laser, 80°C, 21.6 pJ) | **No degradation at 1 Mrad(Si)** gamma |
| **AS216MA1G2B-ASC** (the "failed" one) | **SEL observed**, 21.1 ≤ LET_th ≤ 58.8; SEL at laser ~40 pJ | — |

So "an Avalanche MRAM failed" refers to a *different part number/configuration*, not the family. The part
your decision rests on passed cleanly to **1 Mrad** — orders of magnitude beyond any LEO mission dose.

Key points for the decision:
1. **The Everspin "passed with RP2350" result and the Avalanche "passed at Goddard" result are not in
   conflict** — both MRAM families are good radiation citizens because the magnetic bits are inherently
   immune; the question is always the CMOS periphery and SEL.
2. **Your current argument is real and in Avalanche's favor:** ~50 mA writes vs ~200 mA for the Everspin
   EM016LXQBDH13CS2T. 4× lower write current matters for (a) power budget and (b) — relevantly here —
   **a lower current limit threshold makes SEL detection cleaner**, since a latch-up over-current stands
   out more against a smaller normal operating current.
3. **The caveat:** confirm your exact ordered part number **AS3016204** matches the **ASV016204**
   die/process that Goddard tested (density, process node, and especially that it is not the
   `AS2...MA...-ASC` variant that showed SEL). If the "30" vs "V0" difference is just density/packaging on
   the same 40nm process, you are fine. If it is a different process generation, get that confirmed by
   Avalanche or budget a confirmatory SEL screen. **This is the one open action item.**

**Recommendation: keep Avalanche; verify the part-number/process lineage against ASV016204; keep the SEL
current-limit protection on the MRAM rail regardless (cheap insurance for the CMOS periphery).**

### Q: Tolerance from a part with *no* flight heritage and *no* AEC-Q?

Assume the generic COTS floor and **test or protect, don't trust**:
- TID: commercial silicon generally functional to **~2–10 krad(Si)** minimum; modern deep-submicron CMOS
  usually much more (100 krad → Mrad), older/bipolar/large-geometry parts can fail **<5–30 krad**, and
  **EUV/precision-analog and power-management parts are the ones that fail low** (see analog section).
- SEE: unknown LET_th — must assume it *can* latch. Put it behind a current limiter / switchable rail.
- This is exactly the population your ADM1176 + PMOS architecture exists to protect. For an unknown part in
  a critical slot, either (a) beam-test it, (b) pick a part with data, or (c) make its rail independently
  power-cyclable and current-limited.

### Q: EUV precision-analog subsystem (nA currents, sub-pA resolution) — does radiation drift bias/leakage? Can we compensate?

**Yes, this is your most radiation-*sensitive* subsystem by far, and the effect is real and progressive.**
- **TID increases junction leakage and input bias currents in analog parts**, and it does so *continuously*
  with accumulated dose (not a single event). The TI handbook's bipolar input-bias-current curves
  (Figs 5-5, 5-6, 5-10) show input bias current drifting by **tens to >100 nA over 100 krad**, and
  ELDRS-prone parts drift *worse at the low dose rates you actually see in orbit*. For a front-end trying
  to resolve **sub-pA**, even a few-pA shift in op-amp I_bias or mux leakage is a full-scale problem.
- **Low-leakage muxes/switches are specifically vulnerable**: Goddard data shows e.g. DG409 mux degradation
  **below 1 krad**, while better switches (MAX4595/MAX4651) held to 100 krad. Part selection here swings
  the answer by 100×.
- **Op-amp TID survival is part-specific:** e.g. AD8065 ~30 krad vs RH1014 ~125 krad (Goddard analog table).
  Pick the front-end op-amp and mux on *their TID/I_bias-drift data*, not on noise/bias spec alone.

**Compensation — yes, and you should design for it from day one:**
1. **In-situ dark/zero calibration.** Periodically disconnect the photodiode (you already have low-leakage
   muxes) and measure the front-end offset + leakage with input grounded/shorted. Subtract this measured
   dark current from readings. This removes *both* slow TID drift and temperature drift in one operation —
   it is the single highest-value mitigation for this subsystem.
2. **On-board reference current** (inject a known nA/pA from a stable source through the same path) to track
   gain drift, not just offset.
3. **Choose ELDRS-characterized or known-robust analog** (SiGe BiCMOS and modern thin-oxide parts degrade
   less; large-geometry classic bipolar can be worse — see rules of thumb). ADI and TI both publish
   space-grade analog (e.g. ADA4084/ADA4610, LMP-series) if a COTS part won't hold.
4. **Shield this board's analog front-end** if mass allows — TID (unlike heavy-ion SEE) *is* reducible by
   shielding, and a few krad saved directly extends front-end calibration validity. (Shielding does little
   for heavy-ion SEE — see rules of thumb.)

**Less-critical analog (PGA, ADC, general):** these tolerate more drift. Still prefer parts with TID data;
the ADC's main risk is SET glitches on conversions (mitigate by averaging/median-filtering multiple
conversions) and TID INL/offset drift (caught by the same dark/reference calibration).

### Q: Is our SEL mitigation strategy sound?

**Yes — it is well-aligned with the literature and with proven CubeSat practice. Specific notes:**
- **Detect-and-power-cycle is the textbook COTS-SEL mitigation.** TI handbook §4.6: when SEL-free parts
  aren't available, "adding external circuits can detect the occurrence of an SEL (usually by monitoring the
  supply current ... which increases significantly with an SEL onset) and rapidly initiate a power-down
  reset to minimize damage." Your ADM1176 current monitors do exactly this. 
- **Speed matters — latent damage.** TI §4.6 warns that even SEL events "thought to be nondestructive" can
  cause **latent electromigration damage** that shortens device life. So the limiter must trip *fast* and
  hard. Make sure the ADM1176 over-current trip + PMOS cutoff loop is as fast as practical, and don't rely
  on a slow software poll alone for the first line of defense — the **NyanSat discrete hardware
  watchdog/current-limiter is exactly the right backstop** because it acts without software.
- **Layered architecture is correct:** (1) main ADM1176 system-level limiter, (2) discrete HW
  watchdog+limiter that can restart the whole bus, (3) per-subsystem ADM1176 + RP2350-controlled PMOS
  switches for independent power-cycling. This gives you both whole-system and surgical recovery, plus a
  non-software safety net. ADM1176 flight heritage on PyCubed is a plus.
- **Set trip thresholds from real SEL currents** (next question) with margin above each rail's true max
  operating current. Lower normal operating current ⇒ easier/cleaner SEL discrimination (another reason the
  50 mA Avalanche MRAM helps).
- **Gaps to check:** (a) ensure the PMOS switch + sense path itself can't be the latch victim that defeats
  recovery (the switch should de-power *downstream* of its own controller); (b) make sure cycling a rail
  actually drops V below the SEL **holding voltage** long enough for the parasitic SCR to release (a brief
  brown-out may not clear a latch — fully remove power for a defined off-time).

### Q: Solar battery chargers outside the switches — safe from SEL?

**Mostly yes, with a caveat about the destructive window, and adding PMOS cutoff is worth it.**
- Your reasoning is sound: **if the solar panel's own short-circuit current is below the SEL destructive
  threshold, a latch-up is self-limited** and can't run away to thermal destruction — the panel simply can't
  source enough current. This is a legitimate passive protection (the SEL "power supply can source a current
  greater than the holding current" condition from TI §4.6 is what you're denying).
- **Caveat:** "current-limited" prevents catastrophic burnout but the part is still *latched and
  non-functional* until power is removed, and **latent electromigration damage can still accrue** even at
  modest sustained currents (TI §4.6). A charger stuck latched also stops charging — a mission-availability
  problem even if not a destroyed-part problem.
- **So: add the PMOS cutoff on the solar inputs.** Being able to momentarily open the panel feed to force
  the charger out of latch (and back to normal) costs little and converts "self-limited but possibly stuck
  and slowly degrading" into "recoverable." Recommended.

### Q: Common SEL currents to expect?

Order-of-magnitude data points from the corpus (use to set limiter thresholds, expect wide spread):
- MSP430 (JPL/Guertin): SEL onset around **~50 mA**, ranging up to **0.5 A and beyond** as conditions worsen;
  notably **not recoverable by reset** — required power cycle.
- PIC24/dsPIC33 (JPL): SEL currents **>200 mA**.
- AT22V10B (TI/National AN-989): latch current **~1000 mA (1 A)**.
- General rule: SEL current is "significantly higher" than normal operating current and rises sharply at
  onset — that *step* is what you detect, more than an absolute number.

**Design implication:** size each rail's trip a defined margin above its true worst-case operating current
rather than to a universal number; expect possible latch currents from tens of mA to >1 A depending on part.
Smaller, lower-current loads give you better SEL contrast.

### Q: Rules of thumb (PMOS vs NMOS, shielding, etc.)

- **Lower operating voltage = more SEU, but less SEL and better TID.** (TI AN-989; TI handbook §5.1/§6.1.)
  Running cores/memories at the low end of their voltage range trades a bit more upset for less latch-up and
  more dose margin — generally a *good* trade for a recoverable-SEU / destructive-SEL world.
- **NMOS is the usual TID-leakage culprit, not PMOS.** TID builds positive charge in isolation oxides that
  inverts p-type silicon and opens an **NMOS edge-leakage** path (TI §5.1, §6.3). "Annular/enclosed-gate
  NMOS" is the classic fix in rad-hard parts. So for TID-induced static-leakage growth, NMOS-heavy nodes
  degrade first. (For SEL it's about the parasitic PNPN as a whole, not P vs N alone.) Net: don't assume
  "PMOS = safe," but NMOS off-state leakage is the typical TID failure mechanism.
- **Shielding helps TID and trapped protons/electrons; it does NOT meaningfully stop heavy-ion SEE** (and
  can slightly worsen things via secondary particles). TI Ch.6 opening: for space "shielding the natural
  high-energy particle fluxes is not an option." Use shielding to buy **TID margin** for dose-sensitive
  parts (your EUV analog front-end, any low-TID part); do **not** expect it to fix SEL/SEU. Spot-shield the
  few dose-critical components rather than the whole box.
- **SOI/SOS is latch-up-immune** (no parasitic PNPN path to ground); epitaxial layers and triple-well reduce
  but don't eliminate SEL (TI §6.2). You won't choose process, but it tells you *why* some parts are
  SEL-free and others aren't — and to prefer parts documented on SOI or with guard-rings when SEL data is
  absent.
- **Power MOSFETs: derate V_DS hard.** SEB/SEGR thresholds are well below rated V_DS; trench-gate MOSFETs
  tolerate more than planar, and safe |V_DS| drops as |V_GS| rises (Trench-Gate MOSFET paper; TI §4.7).
  Pick switching FETs with generous voltage margin.
- **Bigger load capacitance suppresses analog SET amplitude/width** (TI Fig 5-13: LM117 SET amplitude drops
  sharply with added output cap). Cheap SET mitigation on regulators/references: add output capacitance.
- **MBU rate ≈ 5–15% of SBU rate** for commercial SRAM/DRAM (TI §4.5/§6.5) — matters for choosing ECC
  strength (SEC-DED is enough only if you also scrub before doubles accumulate).

### Q: Software-side SEU mitigation strategies

From the rad-hard software paper (Stakem) and TI Ch.6, in rough priority for your RP2350:
1. **Memory scrubbing** — periodically read/correct RAM (and re-verify code) on a timer so single-bit upsets
   are fixed before a second upset turns a correctable SBU into an uncorrectable MBU. *This is the highest
   value single measure.*
2. **EDAC / ECC in software** — SEC-DED Hamming over critical RAM regions and over MRAM-stored data. SEC-DED
   corrects 1 bit, detects 2; combined with scrubbing it handles the ~5–15% MBU tail.
3. **Checksum/CRC over code (and config)** held in MRAM; re-verify periodically and on boot; re-load from
   MRAM golden copy on mismatch. (The IUE 1978 case study in the paper: corrupted interrupt vectors caused a
   real anomaly — protect vector tables / critical pointers specifically.)
4. **Watchdog + sanity/self-diagnosis tasks** — detect SEFI/hangs (your HW watchdog already backstops this);
   add a periodic software BIST and a spurious-interrupt sanity check.
5. **Redundant storage of critical state** (e.g. triple-store key variables in MRAM and majority-vote on
   read — software TMR for data; the RP2350's dual cores also enable lock-step-style cross-checks for
   critical computations).
6. **Stack monitor / bounds checks** to catch upset-induced corruption early.
7. **Defensive coding:** validate sensor data ranges, re-init peripherals periodically, design every state
   machine to recover from an illegal state (don't trust a register read once).

### Q: Expected RP2350 RAM error rate from radiation?

No exact per-bit number exists in this corpus specifically for RP2350 SRAM, but you can bound it:
- SRAM SEU cross-sections in the JPL/Guertin compendia are on the order of **~10⁻⁸ to 10⁻⁷ cm²/bit** at
  saturation with low LET_th (commercial SRAM upsets easily). Deep-submicron parts trend lower per-bit but
  pack more bits (TI §6.1, Fig 6-2: per-bit SEU saturated around 180/130nm).
- LEO trapped+GCR flux is low and dominated by **SAA passages** — practical CubeSat experience and the
  derived ISS SEL rates (Guertin: ~2×10⁻⁵ to 4×10⁻⁴ SEL/device/day, ~10× higher for GCR) imply **SEU events
  in a ~520 KB SRAM on the order of a handful per day to low tens per day**, heavily clustered in SAA
  crossings — *not* a continuous storm, but frequent enough that an unscrubbed long-lived variable *will*
  eventually be hit.
- **Therefore: assume RAM bit-flips are a routine, expected event — scrub on a timer, ECC the critical
  regions, and never treat a single RAM read of a critical value as ground truth.** The exact rate matters
  less than designing so any single upset is caught and corrected. (If you want a hard number, the Holliday
  RP2350 beam data is the right source — request σ vs LET from that test.)

### Q: Do we even need to be this concerned about radiation in space?

**Concerned, yes; paranoid, no — and the corpus says exactly where to spend effort.** For a LEO CubeSat:
- **TID is rarely the killer** for modern COTS — your likely mission dose (Part 2) is a few hundred to a few
  thousand rad/yr behind minimal shielding, and most deep-submicron digital parts survive 100 krad → Mrad.
  The exceptions that *will* bite you are **low-TID precision analog (your EUV front-end), power management,
  and any large-geometry/bipolar part** — these can fail in the **single-digit-to-tens-of-krad** range, so
  TID effort should be *targeted* at those, not spread evenly.
- **SEE is the real operational risk**, and it splits cleanly: **SEU/SEFI are frequent but recoverable**
  (handle in software — scrub/EDAC/watchdog), while **SEL is rare but destructive** (handle in
  hardware — current-limit + power-cycle). Your architecture already separates these correctly.
- Multiple sources (MDPI "Towards COTS," EPCI, Airbus) conclude that **well-managed COTS is a sound,
  cost-effective strategy for LEO** — the COTS dose models even *overestimate* real dose by 3–30× (MDPI),
  and screened automotive parts perform comparably to space-grade under derating. The risk is manageable
  *because* you are building the mitigations you listed.

**Net: your concern level and your specific mitigations are appropriate and proportionate. The biggest
real risks for you are (1) SEL on an unprotected/under-protected rail, and (2) TID/leakage drift in the EUV
analog front-end. Everything else is largely covered by scrub + watchdog + current-limit.**

---

## Part 2 — Supporting detail

### 2.1 The LEO radiation environment (your dose budget)

- LEO low-inclination (<30°, 400–600 km): **~100–1000 rad/yr** (Llis-824); per-day figures of **~1–3
  rad/day** (TID-COTS-characterization paper) → ~**0.4–1.1 krad/yr** at the low end.
- Higher inclination / higher altitude: **1000–10,000 rad/yr** (Llis-824). SAA passages dominate trapped
  proton dose and SEE rates.
- **COTS dose models overestimate actual dosimetry by 3–30×** (MDPI "Towards COTS," using ISS measurements)
  — so a model that says "10 krad/yr" likely means a few hundred rad/yr in reality. Don't over-design TID.
- Implication: **a 2–3 year mission behind even a few mm of aluminum is plausibly a few-krad total dose
  budget.** That comfortably clears modern digital CMOS and the Avalanche MRAM (1 Mrad). It does **not**
  automatically clear a 1-krad mux (DG409) or a 30-krad op-amp at end-of-life with margin — hence targeted
  analog part selection + shielding + calibration.

### 2.2 Component-by-component

**RP2350B** — recoverable SEU/SEFI dominate; SEL possible (sub-90nm bulk, no SEL-free guarantee) → keep on a
current-limited, power-cyclable rail; TID not limiting. Use dual-core for critical-compute cross-check; scrub
RAM; CRC the code image in MRAM.

**Avalanche AS3016204 / ASV016204 MRAM** — magnetic bits immune; CMOS periphery is the SEU/SEL surface;
passed to 1 Mrad TID and showed no SEE at Goddard (for ASV016204). Verify part lineage; keep current-limit
on its rail; EDAC the stored data anyway (protects against periphery SEFI during a read/write).

**EUV precision-analog front-end** — *highest TID sensitivity in the system.* Select op-amp and low-leakage
mux on TID/I_bias-drift data (avoid DG409-class early failers); design dark-current/zero auto-calibration
and an injected reference current; consider spot shielding. PGA/ADC: average conversions to kill SETs;
calibrate offset/INL drift.

**Power management (regulators, references, chargers, switches)** — SEL on switching regulators is a real
destructive risk; derate power MOSFET V_DS hard against SEB/SEGR; add output capacitance to references/LDOs
to suppress SETs; put loads behind current limiters; add solar-input PMOS cutoff for the chargers.

### 2.3 SEL mitigation — design checklist

- [x] System-level ADM1176 current limiter (flight heritage).
- [x] Discrete hardware watchdog + current limiter (NyanSat) — software-independent backstop. **Keep this;
      it covers the latency gap where software polling is too slow.**
- [x] Per-subsystem ADM1176 + RP2350-controlled PMOS switch for surgical power-cycling.
- [ ] Verify trip speed is fast enough to avoid latent EM damage (don't rely on slow software poll alone).
- [ ] Verify a power-cycle drops below SEL **holding voltage** for a defined off-time (full off, not brownout).
- [ ] Add PMOS cutoff to solar charger inputs (recover latched-but-current-limited chargers).
- [ ] Set each trip threshold from that rail's true max operating current + margin (use the SEL current data).
- [ ] Confirm the protection/switch path can't be the latch victim that blocks its own recovery.

### 2.4 Mitigation toolbox reference (from TI Ch.6 — what the rad-hard parts do, so you know what you're
emulating in software/architecture)

- **RHBP (by process):** highly-doped substrate / epi to cut SEL; triple-well to cut SEU+SEL; SOI/SOS for
  latch immunity. *(You can't do these — informs part selection.)*
- **RHBD (by design):** guard rings (cut SEL/charge-sharing, ~1.3–1.8× area), upsized transistors (cut SET),
  annular NMOS (kill TID edge leakage), **DICE latches** (orders-of-magnitude SEU reduction), **temporal+
  spatial TMR latches**, **SEC-DED/DEC-TED ECC** with **bit-interleaving (MUX factor)** to turn
  uncorrectable MBUs into correctable SBUs, and **dual/triple-core lockstep**. *(Your software EDAC + scrub +
  data-TMR + dual-core cross-check are the system-level analog of the bottom three.)*

---

## Part 3 — Per-paper summaries (searchable)

**`NASA Goddard Radiation tests.pdf`** (Topper 2020 NEPP Compendium) — *Most important for your MRAM and
analog decisions.* Table III: **ASV016204** (Avalanche 40nm MRAM) no SEE (laser 80°C 21.6 pJ), **no
degradation at 1 Mrad(Si)**; **AS216MA1G2B-ASC** (Avalanche 40nm MRAM) **SEL** 21.1 ≤ LET_th ≤ 58.8, SEL at
laser ~40 pJ. Analog: DG409 mux degrades **<1 krad**; MAX4595/MAX4651 switches good to 100 krad; op-amps
RH1014 ~125 krad, AD8065 ~30 krad.

**`Radiation Test Results for Common CubeSat Microcontrollers...pdf`** (Guertin/JPL 2015) — MSP430F1611/1612:
SEL ~0.05 A onset → 0.5 A+, **not reset-recoverable**; unbiased TID >20 krad, biased fail 5–10 krad.
PIC24/dsPIC33: SEL >200 mA, fail 10–20 krad. AT91SAM9G20: **no SEL observed**. Intel Atom E620T: SEE
crashes. SRAM SEU cross-section tables.

**`Geurtin JPL CubeSat Processor Radiation Efforts.pdf`** (2014 presentation) — companion to above; MSP SEL
not reset-recoverable; **ISS SEL rate ~2×10⁻⁵–4×10⁻⁴/device/day, ~10× for GCR**; SRAM SEU σ tables. Good for
event-rate estimation.

**`NASA Goddard ... / Freescale MRAM Test.pdf`** (MR2A16A 4M MRAM) — first errors at **90–100 krad(SiO2)**;
NASA spec 50 krad with 2× margin; no SEL (not stressed); **magnetic bits SEE-resistant, sensitivity is in
CMOS periphery** — the general MRAM principle behind your part choice.

**`TID characterization of COTS parts.pdf`** — Raspberry Pi CM (BCM2835) functional fail at **66.83 krad**;
LEO 400–600 km / 25–50° ≈ **1–3 rad/day**. Useful Pi-silicon datapoint and dose rate.

**`TI SEU and SEL Considerations for CMOS Devices.pdf`** (National AN-989) — **lower bias ⇒ more SEU, less
SEL**; AT22V10B latch current **~1000 mA**; SEU results don't extrapolate 5V→3.3V. Core "rules of thumb"
source.

**`TI Radiation Handbook for Electronics.pdf`** (~115 pp; read Ch.1,4,5,6) — the master reference. Ch.4: SEE
mechanisms incl. **§4.6 SEL** (detect-current-and-power-cycle; latent EM damage warning), §4.7 SEB/SEGR.
Ch.5: sensitivity by technology (STI vs LOCOS, bias-voltage effect, bipolar ELDRS, **SEL likely <90nm bulk
without guard rings**, input-bias-current drift curves for analog). Ch.6: full mitigation toolbox (epi,
triple-well, guard rings, annular NMOS, DICE, TMR, **SEC-DED/DEC-TED ECC + bit-interleaving**, lockstep).

**`rad hard software.pdf`** (Stakem) — software mitigation catalog: current monitoring as radiation
telltale, self-diagnosis, spurious-interrupt test, **memory test/scrub, checksum-over-code, watchdog, stack
monitor, TMR, EDAC, BIST**. IUE 1978 corrupted-interrupt-vector case study. Primary source for your software
strategy.

**`Trench-Gate MOSFET ... Single Event Effects.pdf`** — trench-gate MOSFETs less SEGR-prone than planar, bear
up to 75 MeV·cm²/mg; **safe |V_DS| decreases as |V_GS| increases** → derate switching FETs.

**`Towards the Use of COTS devices in Space.pdf`** (MDPI 2023) — **COTS models overestimate dose 3–30× vs ISS
dosimetry**; newer thin-oxide CMOS/BiCMOS more TID-robust; FRAM MCU >490 krad unbiased but 7.5 krad in deep
sleep (bias/mode matters!). Best single "is COTS OK?" reference.

**`Llis 824 - Space Radiation Effects ... LEO.pdf`** (NASA Lesson Learned) — LEO <30°: **100–1000 rad/yr**;
higher incl: **1000–10,000 rad/yr**. Commercial **2–10 krad**, SEU LET ~5; rad-tolerant 20–50 krad; rad-hard
>200 krad. **SOI/SOS no latchup.** Best concise tolerance/dose taxonomy.

**`NASA Automotive Component Reliability Studies.pdf`** (Sampson 2016) — AEC-Q grades defined (Grade 1
−40/+125°C); **AEC has no screening requirement and no radiation data**; weigh full cost of ownership. Key
AEC-Q reality check.

**`Commercial vs Space - EPCI.pdf`** — application-characteristics table (space 90% on-time + high radiation
vs automotive 5% on-time + *no* radiation requirement); **automotive parts under space derating ≈
space-qualified failure rate.** Supports a screened-COTS approach.

**`4.3_ADS_Newspace_..._HEDPT_workshop.pdf`** (Airbus) — AEC-Q usable as-is for newspace *reliability*, but
**radiation must be managed per diffusion lot/wafer fab**; tracecode insufficient; **~77% AEC-Q usable at
system level** (80% passives, 25–40% actives, no MOSFET/RF). The "how to actually use automotive parts"
playbook.

**`ADI Challenges-for-Electronic-Circuits-in-Space-Applications.pdf`** — space environment overview + ADI
space-grade analog (ADA4084-2S, ADA4610-2S, ADuM7442S). Use when picking a hardened analog fallback for the
EUV front-end.

---

## Part 4 — Best papers for further research & reference

**Tier 1 — read in full / keep on the bench:**
1. **TI Radiation Handbook for Electronics** — the master reference for mechanisms, technology sensitivity,
   and the full mitigation toolbox. Everything else is a datapoint; this is the framework.
2. **NASA Goddard Radiation tests (Topper 2020 Compendium)** — contains *your* MRAM and analog part data;
   the document your MRAM decision stands or falls on.
3. **Towards the Use of COTS devices in Space (MDPI 2023)** — best modern, evidence-based treatment of
   whether/how COTS works in LEO, with real dosimetry.

**Tier 2 — targeted, high-value:**
4. **Guertin/JPL CubeSat microcontroller results (both PDFs)** — concrete SEL currents, TID-fail, SEU σ, and
   ISS event-rate numbers for MCU-class parts.
5. **rad hard software (Stakem)** — your software-mitigation checklist.
6. **Llis-824 (NASA Lesson Learned)** — the one-page mental model for dose vs tolerance class.

**Tier 3 — reference when the topic comes up:**
7. **Sampson (NASA Automotive)** + **Airbus Newspace HEDPT** + **EPCI** — the AEC-Q/COTS-qualification
   reality (read together).
8. **Freescale MRAM** + **TID-COTS-characterization (Pi)** — supporting MRAM-principle and Pi-silicon
   datapoints.
9. **Trench-Gate MOSFET SEE** + **ADI Challenges** — when designing power switching and selecting hardened
   analog respectively.

**Worth chasing down (not in this corpus, referenced by the brief):**
- **Holliday (NASA) RP2350 + Everspin beam test** — request the actual σ-vs-LET and TID numbers; this is the
  authoritative source for your RP2350 RAM error rate and the Everspin comparison.
- **maholli GitHub discussion (RP2350 vs SAMD51 / Everspin)** — practical PyCubed-community context.
- **Avalanche AS3016204 vs ASV016204 part/process confirmation** — close the one open MRAM action item.
