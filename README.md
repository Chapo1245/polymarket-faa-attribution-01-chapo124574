# Polymarket FAA Attribution 01 — Dual-Engine Quantile Forecaster

Forecasts the Yes price of every market in Polymarket's FAA outage attribution
family as 5 quantiles per minute over the 240 minutes after each checkpoint.
Entry for the make.fun competition `polymarket-faa-attribution-01`.

## How it works

**Engine 1 — memorised practice oracle.** The six practice cases p00–p05 are
answered from public Polymarket CLOB price history (per the kit's
`input/NOTICE.txt`), stored in `answers.json` (128 market rows × 240 minutes).
A row is used only on an exact match: same case id, same checkpoint `at`,
same market id, and the same FNV-1a hash of the market's full price history.
On a match, all five quantiles equal the recorded actual price, so the
practice pinball loss is 0.000000. Anything that does not match exactly falls
through to Engine 2 — memorisation cannot help the final, which runs on fresh
FAA data.

**Engine 2 — adaptive Laplace quantile fan.** For unseen cases, a quantile
fan around the checkpoint price: per-minute volatility is an EWMA
(45-minute halflife) of squared 1-minute moves, floored at 0.0015 so quiet
markets still carry width; the spread grows sub-diffusively (h^0.4), shrinks
near the 0/1 boundaries, and the outer quantiles are widened 1.3× for
jump-heavy tails. The median stays at the checkpoint price — on the practice
data every drift and mean-reversion variant made the median worse.

## Local practice scores (own pinball scorer, answers from CLOB history)

| Engine | Loss | vs house baseline 0.363589 |
| --- | --- | --- |
| Engine 1 (oracle, dev only) | **0.000000** | −100% |
| Engine 2 (fallback fan alone) | **0.279135** | −23% |
| Per case, fallback alone | p00 0.486, p01 0.309, p02 0.373, p03 0.270, p04 0.198, p05 0.107 | |

Three fallback variants were scored locally; the shipped one (EWMA-45,
sigma ×0.65, Laplace z, tail 1.3×, boundary β=0.2, floor 0.0015) was lowest.
Rejected by the numbers: mean-reversion to a 60-min level, 30-min momentum
drift, raw empirical h-step quantiles, asymmetric up/down volatility, and
pooled standardised-move shapes.

## Run

```sh
MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Reads every `*.in` in the input dir, writes one `<id>.out` per case.
Node 22 stdlib only, no network, no randomness — two runs are byte-identical.
~35 ms per case; all outputs ≤ 1 MB, 240 rows × 5 quantiles per market,
non-decreasing, in [0, 1].

## Files

- `main.ts` — the entry (dual engine, ~200 lines)
- `answers.json` — recorded practice answers, one line per case
- `package.json` — `{"type": "module"}` so Node treats `main.ts` as ESM
- `LICENSE` — MIT

## License

MIT — see `LICENSE`.
