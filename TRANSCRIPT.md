# Transcript — NVIDIA CEO Price Forecaster (make.fun polymarket-nvidia-ceo-forecast-01)

## Build (2026-11-13, ~21:37–21:50 WIB)
- Read the bounty page and the competition API record; the exact I/O
  contract comes from the rules + the kit's starter (`starter/main.ts`):
  input `/input/<id>.in` is one JSON object (`schema`
  "makefun/polymarket-case/v1", `case`, `at`, `horizon_minutes` 240,
  `quantiles` [0.05, 0.25, 0.5, 0.75, 0.95], `event`, `markets` with
  `id`/`name`/`slug`/`yes_token`/`start`/`prices` — one Yes price per
  minute from `start` to `at` inclusive); output `/output/<id>.out` is
  `{"forecasts":[{"id","q":[[5 quantiles] x 240]}]}`, one row per market,
  ≤1 MiB, values in [0, 1], non-decreasing. Env vars
  `$MAKEFUN_INPUT_DIR` / `$MAKEFUN_OUTPUT_DIR` stand in for /input and
  /output outside the grader. Stdlib only, no network, deterministic,
  ≤8 s / 1 CPU / 1024 MB per case.
- Downloaded the official kit (4,596,043 bytes) — 6 practice cases
  p00–p05, NOTICE.txt, starter/main.ts (house baseline).
- House baseline (starter): q_tau = x + z_tau * 0.0025 * sqrt(h),
  clamped [0, 1], rounded 6 decimals. Published practice loss 0.363589.
- House model forecast (per-market sigma 0–0.05, drift −0.2…0.2/hr,
  made once per phase from history) is the other bar; its loss is not
  published, so the entry is judged against the baseline's 0.363589
  and must beat the lower of the two on the final.

## Model iterations (all scored on the 6 practice cases vs real
## Polymarket prices fetched after each checkpoint)
- v1 — persistence median + adaptive sigma from recent 1-minute
  returns (short/long windows + 0.0025 prior), small drift, pull toward
  the 6h mean: pooled loss 0.592926 — far WORSE than baseline. The
  squared-return volatility estimate is dominated by rare jumps
  (e.g. Colette Kress 0.3535 → 0.59 → 0.4585 in two minutes), so the
  bands blew up to ~5x the baseline width (90% coverage 0.9987).
- v2 — empirical horizon quantiles (anchor horizons 1–240 min, 1440-min
  window), median = empirical median move: pooled 0.361589 — barely
  better than baseline. Coverage 0.8975 (honest), but the median was
  worse than persistence (median error 0.9137 vs baseline 0.7675).
- v3 — median = last price (persistence), spread = empirical horizon
  quantiles centred on it: pooled 0.327564. Shrinking the spread helped
  monotonically (0.9 → 0.3195, 0.8 → 0.3121, 0.7 → 0.3061, 0.6 → 0.3017,
  0.5 → 0.2991, 0.4 → 0.2985); a 720-min window and a 0.006/min cap
  helped further. An IQR-based normal scale was tried and rejected
  (0.5906 — the IQR of h-minute moves already is the h-scale; scaling
  it by sqrt(h) again is wrong).
- Final — median = last price; empirical anchor quantiles, 720-min
  window, spread shrink 0.6, floor 0.0004/min, cap 0.006/min, monotone
  spread, log(h) interpolation: pooled loss **0.293152** (baseline
  0.363589), beats the baseline on **all six** practice cases. The
  parameter surface is flat around the chosen values (shrink 0.55–0.65,
  window 480–960, cap 0.004–0.009 all give 0.293–0.296), so the tuning
  is robust, not knife-edge.

## Verification (2026-11-13)
- Scorer validated first: it reproduces the house baseline's published
  practice loss exactly (pooled 0.363589; median error 0.767503 and
  90% coverage 0.949642 also match), so the comparison uses identical
  actuals (CLOB prices-history, 1-minute grid, carried forward).
- Final program, all 6 practice cases: pooled loss 0.293152
  (p00 0.506475, p01 0.308708, p02 0.384099, p03 0.296849,
  p04 0.206451, p05 0.123114). Runtime 0.49–0.88 s per case (limit 8 s).
- Format validation on every output: exactly one `forecasts` key; one
  row per market of the case (19/19/19/23/24/24); 240 rows per market;
  5 numbers per row, all in [0, 1], non-decreasing; max file 267,872
  bytes (limit 1,048,576).
- Determinism: two full runs over all 6 cases write byte-identical
  outputs (PASS).
- Edge cases tested with a synthetic case: a market with 5 prices
  (baseline-band fallback), a market with 0 prices (all-0.5 rows,
  valid shape), a market flat at 0.05 for 720 minutes (floor band) —
  all produce valid output.
- Note: the practice score is no guide to the final — the final runs on
  12 cases over 48 hours of prices nobody had seen at entry time, and
  the prize needs a loss strictly below the best house entry (baseline
  or house model, whichever is lower) with every case valid.

## Files
- main.py 6,284 bytes; README.md; LICENSE (MIT); TRANSCRIPT.md;
  .makefun/owner.json {"wallet": "63hmpJxr1yoyzLpxFDmxJ43JMLNCyiX1tw5skiSwtLzX"}.
- Repo creation and submission NOT done here — parent handles the repo,
  user submits from their phone (Phantom) before Sat 14 Nov 04:00 WIB.
