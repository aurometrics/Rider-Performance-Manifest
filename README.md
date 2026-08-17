# Rider Performance Manifest

A weekly rider scoreboard for ando — surfaces the top 10 performing riders
based on a weighted composite score, to support cash-voucher incentives
rather than penalty-based enforcement.

**Live board:** https://aurometrics.github.io/Rider-Performance-Manifest/

---

## How it works

Two Google Sheets — a raw order-event log and a pre-aggregated earnings
sheet — get merged and scored each week by `harmonize.py`, producing a
`data.json` file that the dashboard (`index.html`) reads. The dashboard
itself never needs to change week to week; only `data.json` does.

```
Raw Data sheet  ─┐
                  ├─▶  harmonize.py  ─▶  data.json  ─▶  index.html (GitHub Pages)
Earnings sheet  ─┘
```

---

## Weekly procedure

1. **Download both sheets as CSV.**
   In Google Sheets: `File → Download → Comma Separated Values (.csv)`,
   for both the Raw Data sheet and the Earnings & Metrics sheet. Save them
   anywhere convenient — they don't need to be in this repo.

2. **Run the harmonization script.**
   ```
   python3 harmonize.py <raw_data>.csv <earnings>.csv data.json
   ```
   Order matters: raw data file first, earnings file second. The output
   filename must stay `data.json` — that's the exact name `index.html`
   fetches.

3. **Read the console output.**
   The script prints how many riders were found, how many meet the
   25-order weekly eligibility threshold, and flags any rider with a
   delta time over 60 minutes — worth a manual sanity check before
   trusting that week's numbers (this is how the original Hubert
   Lusangale outlier was caught).

4. **Replace `data.json` in this repo.**
   Overwrite the existing file at the repo root. `index.html` does not
   need to be touched.

5. **Commit and push.**
   ```
   git add data.json
   git commit -m "Week of <date>"
   git push
   ```
   GitHub Pages rebuilds automatically, usually within a minute or two.

Optional: run with a `.csv` output filename instead of `.json` to get a
human-readable version you can open in Excel/Sheets to eyeball before
pushing — only the `.json` version is what the live board reads.

---

## Scoring methodology

Every eligible rider (25+ completed orders that week) gets a **weighted
composite score out of 100**, built from 5 metrics:

| Metric | Weight | Direction |
|---|---|---|
| Orders completed | 25% | Higher is better |
| Rejection + Unacceptance (combined) | 25% | Lower is better |
| Delta time (actual vs. estimated delivery time) | 20% | Lower is better |
| Attendance (days active ÷ days in period) | 15% | Higher is better |
| Rider rating | 15% | Higher is better |

**Why rejection and unacceptance are combined:** both measure the same
underlying behavior — a rider declining to work an assigned order — so
scoring them separately would double-count that dimension. Since both
rates share the same denominator (orders assigned), the combined
`decline_rate = rejection_rate + unacceptance_rate` is an exact sum, not
an approximation. The dashboard's rider drill-down still shows the two
component rates separately for transparency.

**How the score is calculated:**
1. Filter to eligible riders (≥25 completed orders that week)
2. Convert each metric to a percentile rank (0–100) within that week's
   eligible pool
3. Invert the "lower is better" metrics, so 100 always means best
   performance across every metric
4. Multiply each metric's percentile by its weight and sum → composite
   score
5. Rank descending; ties broken by lower decline rate

This is recalculated fresh every week from that week's data — nothing
about a rider's score depends on prior weeks.

---

## Data notes

- **Eligibility threshold:** 25 completed orders/week. Riders below this
  are excluded from ranking (shown in the "below threshold" summary
  count) so small sample sizes can't distort the leaderboard.
- **Phone numbers are deliberately excluded from `data.json`** since this
  repo and the live site are public. They're retained in the CSV output
  (`harmonize.py`'s non-JSON mode) for internal use only — do not commit
  a CSV output to this repo.
- **Reporting period length is computed dynamically** from the actual
  date range in the raw data, not hardcoded to 7 days — so attendance
  percentages stay correct even if a given week's export spans a
  different number of days.
- **Delta time anomalies** (>60 min) are flagged in the console but not
  auto-removed — they're excluded from scoring naturally if the rider
  falls below the order threshold, otherwise worth a manual look.

---

## Files in this repo

| File | Purpose |
|---|---|
| `index.html` | The dashboard. Self-contained (React + Babel via CDN, no build step). Fetches `data.json` at load time; falls back to an embedded sample dataset if `data.json` isn't found (e.g. when previewing the file in isolation). |
| `data.json` | Current week's harmonized, scored-ready data. The only file that changes weekly. |
| `harmonize.py` | Merges the two source sheets and computes all metrics. Run this locally each week — it is not run automatically. |

**Do not commit:** raw sheet CSVs, or any `.csv` output from
`harmonize.py` — both may contain rider phone numbers or other data not
meant for a public repo.

---

## Roadmap

- **Historical archive:** currently `data.json` is overwritten weekly
  with no history retained. Archiving past weeks (e.g.
  `data/2026-07-20.json`) would enable week-over-week trend arrows on
  the leaderboard.
- **Push notifications (Phase 2):** notifying riders of their weekly
  rank/score via SMS or WhatsApp, likely via Africa's Talking. Deferred,
  not urgent — would require a small scheduled job, since GitHub Pages
  can only serve static files and can't send anything on its own.
