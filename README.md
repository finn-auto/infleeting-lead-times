# Infleeting Lead Times Analysis

**Measures actual car infleeting duration by brand and compound, comparing predicted vs real-world performance.**

## Quick Access

| View | Link |
|------|------|
| **Interactive Notebook** | [Open in Google Colab](https://colab.research.google.com/github/finn-auto/infleeting-lead-times/blob/main/infleeting-lead-times.ipynb) |
| **Static View** | [View on GitHub](./infleeting-lead-times.ipynb) |
| **NBViewer** | [nbviewer.org](https://nbviewer.org/github/finn-auto/infleeting-lead-times/blob/main/infleeting-lead-times.ipynb) |

---

## What This Analyzes

This notebook tracks how long cars actually spend in each infleeting phase:

```
Ordered → Produced → Arrived at Compound → Tech Prep Done → Ready to Deliver → In Subscription
```

### Key Questions Answered

1. **How long does each phase take?** — Median and average days per brand/compound
2. **Where are the bottlenecks?** — Which phases cause the longest delays
3. **Are our predictions accurate?** — Expected vs actual lead times
4. **Which compounds are fastest/slowest?** — Performance comparison across locations
5. **Where should we update our estimates?** — Data-driven recommendations

---

## Summary of Findings

### Lead Time Accuracy
- Current hardcoded predictions vs actual performance
- Identifies which brand/compound combinations need recalibration
- Shows systematic patterns (consistently early or late)

### Bottleneck Detection
- Highlights the slowest phase for each brand/compound
- Quantifies outlier-heavy combinations (high avg/median skew)

### Distribution Analysis
- P5–P95 percentile windows for calibrated estimates
- Coverage analysis: what % of cars fall within current predictions

---

## Sections

| # | Section | Business Value |
|---|---------|----------------|
| 1 | Brand/Compound Configuration | Current OEM → compound routing |
| 2 | Lead Times — Avg & Median | Core metrics per phase |
| 3 | Heatmap | Visual comparison across all combinations |
| 4 | Stacked Bar Chart | Total lead time breakdown |
| 5 | Skew Analysis | Identifies outlier-prone routes |
| 6 | Single Car Deep Dive | Trace any individual car's journey |
| 7 | Monthly Trend | Fleet-wide throughput over time |
| 8 | Bottleneck Detector | Worst phase per combination |
| 9 | Expected vs Actual | Prediction accuracy analysis |
| 10 | Skip Analysis | Which states get skipped (e.g., tech prep) |
| 11 | Distribution Analysis | Percentile windows for calibration |
| 12 | Granularity Comparison | Brand-only vs Brand+Compound accuracy |
| 13 | Data-Driven Recommendations | Proposed lead time updates |

---

## Data Sources

- **BigQuery**: `datawarehouse-304513.cars_ops_postgres_public.car_events`
- **Time Range**: 2025 calendar year
- **Scope**: All brand/compound combinations in the ops-cars lead times config

---

## For Engineers

### Running Locally

```bash
# Install dependencies
pip install google-cloud-bigquery pandas plotly

# Authenticate
gcloud auth application-default login

# Run notebook
jupyter lab infleeting-lead-times.ipynb
```

### Key Queries

The notebook uses the `car_events` table with these event names:
- `car_ordered`
- `car_state_changed_produced`
- `car_state_changed_arrived_from_supplier`
- `car_state_changed_tech_prep_done`
- `car_state_changed_ready_to_deliver`
- `car_state_changed_in_subscription`

Phase duration = `DATE_DIFF` between consecutive state changes per `car_id`.

### T-Minus Interpretation

The reference table uses **countdown values** (days remaining until handover), not additive phase durations:

```
Phase duration = t_minus[current_state] - t_minus[next_state]
```

Example for BMW at compound with 14-day compound lead time:
- `arrived_from_supplier`: 14 days to handover
- `tech_prep_done`: 10 days to handover → arrived→techprep = **4 days**
- `ready_to_deliver`: 5 days to handover → techprep→ready = **5 days**

---

## Related Resources

- [ops-cars API docs](https://github.com/finn-auto/redocly)
- [Lead Times Reference Table](https://docs.google.com/spreadsheets/d/...) *(internal)*
- [Fleet Operations Dashboard](https://finn.cloud.looker.com/...) *(Looker)*

---

## Questions?

Reach out in **#ops-cars** on Slack.
