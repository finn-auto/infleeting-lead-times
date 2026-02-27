# Infleeting Lead Times — Analysis Findings

**Date**: February 2026
**Authors**: Jan Heiselbetz, Leon Schulz (with input from Lena / Infleeting)
**Notebook**: `infleeting-lead-times.ipynb`
**Data**: BigQuery `cars_ops_postgres_public.car_events` (2025 fleet, 48,549 car transitions across 40 brand/compound combos)
**Code**: `ops-cars-repo/src/domain/availabilities/lead-times/*.ts`

---

## 1. Status Quo — How the System Works Today

### Two-layer architecture

The ops-cars API calculates `Available From` (the date we tell customers and compounds a car will be ready) using two independent systems:

```
Available From = start_date + compound_lead_time
```

| Car Status | start_date | compound_lead_time |
|---|---|---|
| `ordered` / `produced` | `max(Fleet ETA, today)` | `OrderedLeadTimes` / `ProducedLeadTimes` |
| `arrived_from_supplier` | `today` (or anchor date) | `ArrivedFromSupplierLeadTimes` |
| `tech_prep_done` | `today` (or anchor date) | `TechPreparationDoneLeadTimes` |
| `ready_to_deliver` | `today` (or anchor date) | `ReadyToDeliverLeadTimes` |

**Layer 1 — Fleet ETA** (Fleet responsibility): Predicts when the car physically arrives at the compound. Based on OEM Delivery Plans (often just monthly batch estimates via G-Sheet). Covers `ordered` → `arrived_from_supplier`. Transfer of risk from OEM to Finn typically happens at compound arrival.

**Layer 2 — Compound Lead Time** (Infleeting responsibility): Predicts how long the compound needs to process the car (PDI, registration, finish, handover). Covers `arrived_from_supplier` → `in_subscription`. This is what the hardcoded values in `arrived-from-supplier-lead-times.ts`, `tech-preparation-done-lead-times.ts`, and `ready-to-deliver-lead-times.ts` control. Infleeting's goal: *"das Auto fertig zu bekommen zum Available From"* (get the car ready by the Available From date).

### Key design decision: ordered = produced for most brands

The API gives **identical** compound lead times for `ordered` and `produced` status for nearly every brand:

| Brand (Process) | t_ordered | t_produced | Same? |
|---|---|---|---|
| Hyundai BLG | 42d | 42d | Yes |
| FCA (Fiat/Jeep/Alfa) | 21d | 21d | Yes |
| Stellantis (Peugeot/Citroen/DS/Opel) | 28d | 28d | Yes |
| BMW | 45d | 45d | Yes |
| Renault/Dacia | 35d | 35d | Yes |
| Mercedes | 28d | 28d | Yes |

**Only Kia** (56d ordered → 42d produced) and **Nissan** (35d → 21d) have different values.

This means the API assumes **instant production** (0 days in ordered→produced phase) for most brands. The real OEM production/delivery time is supposed to be captured by the Fleet ETA — but as Lena confirmed, Fleet ETAs are often inaccurate or missing, especially for Stellantis.

### Lifecycle states and what they actually map to

Per Lena's walkthrough (Hyundai @ BLG Bremerhaven example):

| Lifecycle State | Real-world meaning | Data trigger |
|---|---|---|
| `ordered` | Car in system, contract with OEM, may be online & bookable. No VIN possible. | Car creation by Fleet |
| `produced` | Car produced, but not necessarily at compound | OEM status update |
| `arrived_from_supplier` | Physical compound arrival (Fin Gate In). PDI starts. | Compound reports arrival |
| `tech_prep_done` | PDI complete + registration done (~9 days before handover) | PS Team confirms registration |
| `ready_to_deliver` | Stickers, plates, SA number, car at pickup spot (~4 days before handover) | Compound finish complete |
| `in_subscription` | Driver picked up the car | Handover confirmed |

---

## 2. What We Did

### 2.1 Built the analysis notebook

- Queried BigQuery for all state transitions of 48,549 cars across 40 brand/compound combinations (2025 data)
- Computed actual median and average days per phase
- Built heatmaps, stacked bars, skew analysis, bottleneck detection, and trend charts

### 2.2 Extracted actual API values from code

Instead of using the manually compiled ops reference spreadsheet, we read the actual TypeScript source files:
- `ordered-lead-times.ts` — per InfleetingProcess, with BMW supplierStatus logic
- `produced-lead-times.ts` — nearly identical to ordered for most processes
- `arrived-from-supplier-lead-times.ts` — conditional on paid/unpaid, compound-specific MG logic
- `tech-preparation-done-lead-times.ts` — Stellantis=9d, default=10d, AUDI/Mazda unpaid=28d
- `ready-to-deliver-lead-times.ts` — highly compound-specific with SA number, outbound check, payment conditions

Found several discrepancies between the ops reference sheet and actual code (see section 3.2).

### 2.3 Corrected the comparison methodology

Initial analysis compared hardcoded values as if they were phase durations. After reading the code and Lena's interview, we discovered:
- The values are **t-minus countdowns** (remaining days until subscription), not phase durations
- The meaningful comparison is **t_arrived vs actual arrived→subscription** (compound processing time)
- Ordered→Produced "misses" are ETA accuracy problems, not compound lead time errors

### 2.4 Interview with Infleeting (Lena)

Key quotes and insights:
- *"Wir nehmen Autos online, auch wenn sie noch nicht produziert sind [...] die sind noch nicht mal in Produktion eingegangen, wir haben keine VIN"* — Cars go online extremely early, especially Stellantis
- *"Mit einer Deviation von acht Wochen online genommen, weshalb das Window so krass lang ist"* — Huge customer windows as a band-aid
- *"ETA und tatsächlicher Delivery Plan driften häufig auseinander"* — Fleet ETAs are unreliable
- *"PDI muss überall gemacht werden [...] manchmal können wir einen eigenen Prozessschritt machen, manchmal nicht, je nachdem ob wir die Daten haben"* — Sub-steps within compound phases lack data granularity

---

## 3. What We Found

### 3.1 Compound lead time accuracy (the meaningful metric)

Comparing API `t_arrived` (predicted days from compound arrival to subscription) against actual medians:

- **Overall**: Only ~40% of compound lead times are within ±20% of reality
- **Overestimated** (cars faster than predicted): Common for arrived→techprep phase — many brands complete PDI faster than the buffer allows
- **Underestimated** (cars slower than predicted): Common for ready→subscription — cars wait longer for subscriber matching than the 5-9 day prediction

### 3.2 Discrepancies between ops reference sheet and actual API code

| Brand/Compound | Sheet Value | Actual Code | Delta |
|---|---|---|---|
| BMW/MINI @ akb_kitzingen | t_ordered = 70d | 45d (supplierStatus 170 default) | -25d |
| Honda @ akb_kitzingen | t_ordered = 21d | 35d (MKM_HUBER = 5w) | +14d |
| Polestar @ cat_zuelpich | t_techprep = 14d | 7d (POLESTAR = 1w) | -7d |
| All Stellantis | t_techprep = 14d | 9d (STELLANTIS = 9d) | -5d |
| All FCA/Hyundai/default | t_techprep = 14d | 10d (default = 10d) | -4d |
| Audi @ akb_kitzingen | t_techprep = 28d | 21d (AUDI unpaid = 3w) | -7d |

**The ops reference sheet is out of sync with the deployed code.** Neither is fully correct — both need recalibration against actual data.

### 3.3 Massive skew in early phases

The avg/median ratio reveals data quality issues:

| Brand | Compound | Phase | Avg | Median | Skew |
|---|---|---|---|---|---|
| Nissan | blg_neuss | Ordered→Produced | 29.6d | 1d | 29.6x |
| Hyundai | mosolf_cuxhaven | Ordered→Produced | 39.4d | 2d | 19.7x |
| Citroen | mosolf_kippenheim | Produced→Arrived | 14.4d | 1d | 14.4x |
| BMW | akb_kitzingen | Arrived→Tech Prep | 13.9d | 2d | 7.0x |

These indicate that most cars skip the phase instantly (median ~0-2d) but a significant minority gets stuck for weeks/months — likely due to financing blockers, call-off problems, or lifecycle state misuse.

### 3.4 Lifecycle state skipping

Several brands routinely skip `tech_prep_done`:
- **MG, BYD**: Often skip tech_prep (noted in API comments)
- **BMW/MINI**: Often skip tech_prep (PDI done at OEM/Garching, not at Finn compound)

When a state is skipped, the API's lead time for that state becomes meaningless — the car jumps directly from `arrived` to `ready`, making the `t_techprep` value irrelevant.

### 3.5 Ready→Subscription is universally underestimated

Across nearly all brands, the actual median for ready→subscription exceeds the API prediction:
- API typically predicts 5-9 days (working days converted)
- Reality is often 8-29 days median
- This is a **demand/matching problem**, not a logistics problem — cars are physically ready but waiting for a subscriber

### 3.6 The Stellantis problem (Lena's "aufregen")

Stellantis brands (Fiat, Jeep, Alfa Romeo, Peugeot, Citroen, DS, Opel) have the worst predictions because:
1. Cars go online in `ordered` with no VIN and no production date
2. Fleet ETA is often missing or based on vague monthly batch estimates
3. With no ETA, `max(ETA, now)` defaults to `now`, making `Available From = today + 21d` — absurd for a car that hasn't entered production
4. Actual ordered→produced median: **69-256 days** depending on brand
5. Result: massive customer windows (8+ weeks deviation) and blocker alerts for cars that aren't even produced

---

## 4. What We Can Improve

### 4.1 Short-term: Recalibrate compound lead times in the API

**Impact: High | Effort: Low**

Update `arrived-from-supplier-lead-times.ts`, `tech-preparation-done-lead-times.ts`, and `ready-to-deliver-lead-times.ts` with data-driven values from the notebook analysis. These are direct code changes to static values.

Priority updates (ranked by impact = error × volume):
1. Brands where `t_arrived` is significantly off from actual arrived→subscription median
2. `ready-to-deliver-lead-times.ts` values — universally too optimistic, increase by ~5-10 days
3. Remove or reduce `t_techprep` for brands that skip tech_prep (MG, BYD, BMW/MINI)

### 4.2 Short-term: Sync ops reference sheet with code

**Impact: Medium | Effort: Low**

The Google Sheet used by ops and the actual deployed code have diverged. Either:
- Auto-generate the sheet from code (preferred), or
- Add a CI check that validates sheet values against code

### 4.3 Medium-term: Separate ETA accuracy from compound lead time accuracy

**Impact: High | Effort: Medium**

The current system conflates two independent predictions. To improve:
1. **Track Fleet ETA accuracy** separately: compare ETA at car creation vs actual `arrived_from_supplier` date
2. **Flag cars with no/stale ETA**: If `max(ETA, now) = now` for an ordered car, the Available From is pure guesswork — surface this to ops
3. **Recalculate Available From when ETA updates**: Currently unclear if this happens consistently

### 4.4 Medium-term: Add percentile-based deviation windows

**Impact: High | Effort: Medium**

Replace the current global deviation with per-brand/compound/phase windows calibrated to historical data:
- P50 (median) for the Available From estimate
- P75 as the "likely" window end
- P95 as the "almost certain" window end
- Target: 95-99% of handovers within the communicated customer window

The notebook has the query infrastructure for this (percentile analysis sections).

### 4.5 Medium-term: Handle state-skipping brands explicitly

**Impact: Medium | Effort: Medium**

For brands that routinely skip `tech_prep_done` (MG, BYD, BMW/MINI):
- Set `t_techprep = t_ready` (or close to it) so the API doesn't add phantom processing time
- Or: add explicit skip-detection logic that adjusts the lead time when a state is likely to be skipped

### 4.6 Long-term: ML model for time-to-ready / time-to-handover

**Impact: Very High | Effort: High**

As discussed with Lena — build a predictive model using:
- **Features**: OEM status, brand, compound, material delays (if available), payment status, historical ETA accuracy per OEM, seasonality, current compound backlog
- **Target**: `actual_available_from` (or `time_to_in_subscription`)
- **Benefit**: Replaces both the static compound lead times AND the unreliable Fleet ETA with a single, continuously calibrated prediction

Prerequisites:
1. Clean lifecycle state mapping per brand (this analysis)
2. Structured material delay data (currently verbal/email only per Lena)
3. Historical ETA vs actual arrival data

### 4.7 Long-term: Granular sub-milestones within compound processing

**Impact: Medium | Effort: High**

Lena noted that within `arrived_from_supplier` → `ready_to_deliver`, multiple sub-steps happen with no individual data points:
- PDI (software, technical checks)
- Tire swap (depends on OEM material delivery)
- Washing
- Storage / Charging
- Registration (PS Team, ~9 days before handover)
- Finish (stickers + plates, ~4-5 days)

Adding structured tracking for even 2-3 of these (e.g., PDI complete, tires done, registration triggered) would dramatically improve mid-process predictions. However, this requires compound cooperation and system changes.

---

## Appendix: Data Sources & Tools

| Source | Purpose |
|---|---|
| `datawarehouse-304513.cars_ops_postgres_public.car_events` | Historical state transitions |
| `ops-cars-repo/src/domain/availabilities/lead-times/*.ts` | Deployed lead time logic |
| `Q4_2025_Infleeting Lead Times - Lead Times.tsv` | Ops reference sheet (partially outdated) |
| Granola meeting notes (Infleeting INTRO Lena, Feb 26 2026) | Process understanding |
| `infleeting-lead-times.ipynb` | Analysis notebook with all queries and charts |
