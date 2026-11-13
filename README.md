# NVIDIA CEO Price Forecaster (make.fun)

A forecaster for the make.fun bounty "Forecast Polymarket's next NVIDIA
CEO prices minute by minute, 4 hours ahead". Given one case — every
trading market of Polymarket's "Next CEO of NVIDIA?" event with its Yes
price at every minute of the 72 hours up to a checkpoint — it writes five
quantiles (5%, 25%, 50%, 75%, 95%) of each market's price at every minute
of the 240 minutes after the checkpoint.

## Model

- **Median: the last price (persistence).** Prediction-market prices are
  (near-)martingales; on the six practice cases any drift estimate — even
  the empirical median move — made the median worse.
- **Spread: empirical horizon quantiles.** For anchor horizons
  [1, 2, 5, 10, 20, 40, 80, 160, 240] minutes, the distribution of actual
  h-minute price moves over the last 12 hours is read off at the five
  quantile levels; the band is centred on the last price and interpolated
  in log(h) between anchors. These prices are mostly flat with rare jumps
  that usually revert, so long-horizon moves are much tighter than
  1-minute moves scaled by sqrt(h) — the empirical anchors capture that
  directly, where the house baseline's fixed 0.0025 sigma does not.
- The empirical spread is shrunk by 0.6 (overlapping move samples and
  rare jumps inflate the raw empirical tails). The band never narrows as
  the horizon grows, and never goes below a 0.0004/min floor or above a
  0.006/min cap (both scaling with sqrt(h)).
- Quantiles are clamped to [0, 1], rounded to 6 decimals and
  non-decreasing by construction.

## Layout

- `main.py` — the forecaster (6,284 bytes). Python 3 stdlib only; no
  network, no other files read or imported. Reads
  `$MAKEFUN_INPUT_DIR` (else `/input`) and writes `$MAKEFUN_OUTPUT_DIR`
  (else `/output`), one `<id>.out` per `<id>.in` case.
- `LICENSE` — MIT.
- `TRANSCRIPT.md` — build log and test results.
- `.makefun/owner.json` — links the repo to the make.fun account wallet.

## Run

```sh
mkdir -p out
MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

## Practice score

Six practice cases (p00–p05), scored against real Polymarket prices
after each checkpoint (CLOB `prices-history`, 1-minute grid):

| case | this entry | house baseline |
|------|-----------|----------------|
| p00 | 0.506475 | 0.536138 |
| p01 | 0.308708 | 0.380352 |
| p02 | 0.384099 | 0.408408 |
| p03 | 0.296849 | 0.368599 |
| p04 | 0.206451 | 0.298794 |
| p05 | 0.123114 | 0.238232 |
| **pooled** | **0.293152** | **0.363589** |

Mean pinball loss in percentage points (lower is better). The entry
beats the house baseline on all six practice cases. The scorer
reproduces the house baseline's published practice loss (0.363589)
exactly, so the comparison is on identical actuals.
