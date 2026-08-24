# Rider Performance Manifest

A rider scoreboard for ando — tracks performance **weekly** for visibility
and motivation, and rolls that up into a **monthly** view that determines
the actual cash-voucher payout (to fit monthly cash flow, not weekly).

**Weekly tracker:** https://aurometrics.github.io/Rider-Performance-Manifest/
**Monthly payout board:** https://aurometrics.github.io/Rider-Performance-Manifest/monthly.html

---

## How it works

Two Google Sheets — a raw order-event log and a pre-aggregated earnings
sheet — get merged and scored each week by `harmonize.py`, producing a
dated file archived under `data/`. Once a month, `aggregate_monthly.py`
combines that month's weekly files into one monthly dataset, using the
same folder. Both dashboards read from `data/manifest.json` to know what
weeks and months are available.

```
Raw Data sheet  ─┐
                  ├─▶ harmonize.py ─▶ data/2026-MM-DD.json ─┐
Earnings sheet  ─┘                                          │
                                                              ├─▶ manifest.json
        (once a month, after 4-5 weekly files exist)         │
        data/week1.json + week2.json + ... ─▶                │
              aggregate_monthly.py ─▶ data/2026-MM.json ─────┘

index.html   (weekly tracker)   reads data/manifest.json → data/<week>.json
monthly.html (monthly payout)   reads data/manifest.json → data/<month>.json
```

**Why two separate pages instead of one with a toggle:** the weekly board
and the monthly payout board serve different purposes and audiences —
weekly is for ongoing visibility, monthly is what actually determines who
gets paid. Keeping them as separate URLs makes it unambiguous which one
you're looking at.

---

## Repo structure

```
your-repo/
├── index.html        (weekly tracker — the default live URL)
├── monthly.html       (monthly payout board)
└── data/
    ├── manifest.json  (auto-maintained list of available weeks + months)
    ├── 2026-07-27.json   (one file per week)
    ├── 2026-08-03.json
    ├── ...
    └── 2026-07.json      (one file per aggregated month)
```

Both `index.html` and `monthly.html` are self-contained (React + Babel via
CDN, no build step) and never need to be edited week to week — only the
`data/` folder changes.

---

## Weekly procedure (every week)

1. **Download both sheets as CSV.**
   In Google Sheets: `File → Download → Comma Separated Values (.csv)`,
   for both the Raw Data sheet and the Earnings & Metrics sheet.

2. **Run the harmonization script, output straight into `data/`:**
   ```
   python3 harmonize.py <raw_data>.csv <earnings>.csv data/2026-MM-DD.json
   ```
   Order matters: raw data file first, earnings file second. Name the
   output file after the week's end date (e.g. the Sunday), so files sort
   naturally.

3. **Read the console output.** It reports total/eligible riders and
   flags any delta time over 60 minutes worth a manual check.

4. This automatically **creates or updates `data/manifest.json`** with
   this week's entry — no manual bookkeeping needed. If you re-run the
   same week's dates, it replaces that week's entry rather than
   duplicating it.

5. **Push the new file plus the updated manifest:**
   ```
   git add data/2026-MM-DD.json data/manifest.json
   git commit -m "Week of <date>"
   git push
   ```
   The weekly tracker (`index.html`) picks up the new week automatically
   — it always defaults to the most recent, with older weeks available
   in the selector dropdown once more than one exists.

---

## Monthly procedure (once a month, after that month's weeks are done)

1. **Confirm all of that month's weekly files exist in `data/`** —
   e.g. four or five `data/2026-07-*.json` files for July.

2. **Run the aggregator, listing every weekly file for that month, with
   the output filename last:**
   ```
   python3 aggregate_monthly.py data/2026-07-06.json data/2026-07-13.json data/2026-07-20.json data/2026-07-27.json data/2026-07.json
   ```
   This does **not** average the weekly percentages together — it sums
   the underlying raw counts (orders, rejections, unacceptances, days
   worked) across all included weeks and recomputes every rate fresh
   from those totals, exactly like `harmonize.py` does for a single
   week. See *Scoring methodology* below for why this matters.

3. This also updates `data/manifest.json` with a `months` entry, and
   automatically sets that month's **eligibility threshold** to
   `25 × number of weeks included` — a 5-week month requires more
   orders than a 4-week one to qualify.

4. **Push the new monthly file plus the updated manifest:**
   ```
   git add data/2026-07.json data/manifest.json
   git commit -m "July 2026 payout aggregate"
   git push
   ```
   `monthly.html` picks it up automatically. This is the board to use
   when determining who receives the voucher.

---

## Scoring methodology

Every eligible rider gets a **weighted composite score out of 100**,
built from 5 metrics:

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
an approximation. Both dashboards' rider drill-downs still show the two
component rates separately for transparency.

**How the score is calculated:**
1. Filter to eligible riders (weekly: ≥25 completed orders; monthly:
   ≥25 × weeks included)
2. Convert each metric to a percentile rank (0–100) within that period's
   eligible pool
3. Invert the "lower is better" metrics, so 100 always means best
   performance across every metric
4. Multiply each metric's percentile by its weight and sum → composite
   score
5. Rank descending; ties broken by lower decline rate

**Monthly scores are not an average of weekly scores.** A rider's
monthly percentile rank is computed fresh from that month's totals — this
is deliberate. Averaging weekly percentages would distort riders with
uneven weeks (e.g. one bad week among three good ones shouldn't average
out identically to a rider who was steady but mediocre all month).
Summing the real counts and recomputing is the statistically correct
approach.

---

## Data notes

- **Weekly eligibility threshold:** 25 completed orders. **Monthly
  threshold:** auto-scales to `25 × weeks included` in that month.
- **Phone numbers are deliberately excluded from all JSON output**
  (weekly and monthly) since this repo and site are public. `harmonize.py`
  retains phone numbers only in its CSV output mode, for internal/local
  use — never commit a CSV output to this repo.
- **Reporting period length is computed dynamically**, not hardcoded —
  weekly from the actual date range in the raw sheet, monthly from the
  sum of each included week's period. This keeps attendance percentages
  correct even if a week's export spans an unusual number of days.
- **Delta time anomalies** (>60 min) are flagged in `harmonize.py`'s
  console output but not auto-removed — worth a manual look, especially
  if the rider is otherwise eligible.
- **Raw counts (not just rates)** are stored in every weekly file
  specifically so monthly aggregation can sum exact numbers rather than
  reverse-engineer them from rounded percentages.

---

## Files in this repo

| File | Purpose |
|---|---|
| `index.html` | Weekly tracker. Reads `data/manifest.json`, defaults to the most recent week, lets you select older weeks once more than one exists. |
| `monthly.html` | Monthly payout board. Same idea, but for months — this is what determines the voucher recipients. |
| `harmonize.py` | Run weekly. Merges the two source sheets, computes all metrics, archives the result under `data/`, and updates `manifest.json`. |
| `aggregate_monthly.py` | Run monthly. Combines a month's worth of weekly files into one correctly-recomputed monthly dataset, updates `manifest.json`. |
| `data/manifest.json` | Auto-maintained index of every archived week and month. Both dashboards read this to populate their selectors. |
| `data/*.json` | The actual weekly and monthly datasets. |

**Do not commit:** raw sheet CSVs, or any `.csv` output from
`harmonize.py` — both may contain rider phone numbers or other data not
meant for a public repo.

---

## Roadmap

- **Historical trend arrows:** now that weekly files are archived instead
  of overwritten, week-over-week rank comparison (▲/▼) is feasible to add
  to `index.html` — not yet implemented, but the data foundation for it
  now exists.
- **Push notifications (Phase 2):** notifying riders of their rank/score
  via SMS or WhatsApp, likely via Africa's Talking. Deferred, not urgent
  — would require a small scheduled job, since GitHub Pages can only
  serve static files and can't send anything on its own.
