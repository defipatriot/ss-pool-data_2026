# ss-pool-data_2026

Daily Skeleton Swap pool snapshots (TVL, volume, APR) for all phoenix-1 pools.

Companion cron: [`defipatriot/cron-scripts/skeletonswap-lp_data`](https://github.com/defipatriot/cron-scripts/tree/main/skeletonswap-lp_data)

---

## Directory layout

```
ss-pool-data_2026/
├── README.md                          ← you are here
├── day-1.csv                          ← rolling daily files (Mon=1, Sun=7)
├── day-2.csv                          ←   overwritten each week
├── ...
├── day-7.csv
├── 6-day-avg.csv                      ← computed Saturday end-of-day
└── data/
    ├── weekly-avg/
    │   ├── 2026-epoch-184.csv         ← Mon end-of-day rollup (full epoch)
    │   └── ...
    └── monthly-avg/
        ├── 2026-05.csv                ← 1st-of-month rollup (full prior month)
        └── ...
```

### Why the day-N rolling pattern

Each day of the week writes to `day-{N}.csv` where N is the ISO day of week (Monday=1, Sunday=7). Files get overwritten week-over-week — Monday May 12 writes to `day-1.csv`; the following Monday May 19 overwrites the same file.

This gives a constant 7-day window of fresh data without accumulating thousands of files. For historical lookback, the cron also writes weekly aggregates to `data/weekly-avg/2026-epoch-{N}.csv` — these are immutable, one per epoch.

The `6-day-avg.csv` is a Saturday-night preview file containing Mon-Sat means, useful for dashboards that want a Sunday read-only of the in-progress epoch.

---

## Refresh cadence

| File | Updated when |
|---|---|
| `day-{N}.csv` | Daily at 23:45 UTC (overwrites same-named file every week) |
| `6-day-avg.csv` | Saturday 23:45 UTC (covers Mon-Sat of current week) |
| `data/weekly-avg/2026-epoch-{N}.csv` | Monday 00:05 UTC (covers prior Mon-Sun epoch) |
| `data/monthly-avg/YYYY-MM.csv` | 1st of month 00:10 UTC (covers prior month) |

---

## Schema — `day-{N}.csv` and weekly/monthly CSVs

```csv
date,time,pool_id,pool_address,tvl_usd,volume_24h_usd,volume_7d_usd,apr_7d,reserve_0,reserve_1,total_share
2026-05-11,23:59:17,"ampLUNA-LUNA",terra1tsx0...,68015.52,,,,423048817403,860454921759,497763224658
```

| Column | Type | Description |
|---|---|---|
| `date` | YYYY-MM-DD | Date of the snapshot (UTC) |
| `time` | HH:MM:SS | Time of the snapshot (UTC) |
| `pool_id` | string | Human-readable pool name from Skeleton Swap API |
| `pool_address` | bech32 | Terra contract address |
| `tvl_usd` | number | Total value locked in USD |
| `volume_24h_usd` | number | 24-hour swap volume |
| `volume_7d_usd` | number | 7-day rolling volume |
| `apr_7d` | number | 7-day swap-fee APR (percentage) |
| `reserve_0` | int | Token 0 raw reserve (micro units) |
| `reserve_1` | int | Token 1 raw reserve (micro units) |
| `total_share` | int | LP token total supply |

Empty fields mean Skeleton Swap's API returned null/missing for that value.

---

## What's captured

**All Skeleton Swap pools on phoenix-1**, not just TLA-relevant ones. As of writing this is ~25 pools. The cron pulls the full pool list from Backbone Labs' aggregator API and doesn't filter — every pool is captured even if it has $0 TVL.

The TLA tool and dashboard filter to TLA-relevant pools at display time using the same canonical naming used by `astroport-pool-data_2026`.

---

## Data source

- **Backbone Labs Skeleton Swap aggregator** at `dex.warlock.backbonelabs.io/api/pools/phoenix-1`

Single API call per cron run. Backbone returns pre-computed TVL and volume so the cron doesn't need any chain queries.

---

## Cross-references

- Pool addresses are usable as join keys with `astroport-pool-data_2026` and `votion-data_2026`
- Some pools exist on both DEXes (e.g. LUNA-ampLUNA) — same display name, different `pool_address`
- Pool naming convention is **NOT LUNA-first** in this repo — Skeleton Swap names them in whatever order the API returns. Renames happen at display time via the canonicalization function used by the website.

---

## Known issues

- **No retry on Backbone API failure** — single API outage means one missed snapshot. Self-heals next day. Tracked for future improvement.
- **Naming inconsistency vs Astroport** — `ampLUNA-LUNA` here vs `LUNA-ampLUNA` in astroport-pool-data. Website handles canonicalization.
