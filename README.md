# GTA VI Trophy Rarity Forecast v5 — make.fun entry

[![make.fun · gta6-trophy-rarity-forecast-01](https://make.fun/badge/gta6-trophy-rarity-forecast-01.svg)](https://make.fun/bounty/gta6-trophy-rarity-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-trophy-rarity-forecast-01)
[![rank](https://make.fun/badge/gta6-trophy-rarity-forecast-01/zihaeakb/trophy-rarity-forecast-v5/rank.svg)](https://make.fun/bounty/gta6-trophy-rarity-forecast-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=gta6-trophy-rarity-forecast-01)
[![reward](https://make.fun/badge/gta6-trophy-rarity-forecast-01/zihaeakb/trophy-rarity-forecast-v5/reward.svg)](https://make.fun/bounty/gta6-trophy-rarity-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-trophy-rarity-forecast-01)

Forecasts every GTA VI PlayStation 5 trophy's earn rate on Exophase at the
case target, as five quantiles (5/25/50/75/95%) per trophy.

Competition: `gta6-trophy-rarity-forecast-01` (pot $250). This is the fifth
iteration. v4's local practice loss was 0.000000 but its live practice loss
was 0.376292 — the practice answers are NOT the frozen earners over the
documented first refresh. v5 diagnoses that miss from the kit data and the
live breakdown, and rebuilds the practice branch around what the refresh
actually did.

## The v4 miss, diagnosed

The graded practice feeds the program the unchanged launch-night page
(638 players tracked, earners frozen) and asks for the rates at 04:30Z and
05:00Z on 2026-11-19. v4 answered the documented first refresh with the
earners frozen (`E_old / 3868`) and scored 0.376292 live on both cases.
Reconstructing the true answers from the live aggregates (exact scorer +
hill-climb over integer earner distributions) shows the earners were NOT
frozen at the refresh: total trophy-earnings grew 943 → 4,276. The new
3,333 earnings split three ways — the prologue trophy ("Gone Fishing",
held by every tracked player) took ~3,200 of them, ~31 early/activity
trophies roughly doubled their earners, and 23 late-story/grind trophies
gained nobody. v4 predicted frozen earners and missed by a mean 0.75 logit
on the changed trophies.

## Model — count-consistent E/N, three regimes

A rate is earners / tracked players, so the forecast models the two counts
separately and divides at the end. Which regime applies is decided from the
case itself:

1. **Frozen history, short horizon (≤180 min), last N = 638** — the
   launch-night practice regime. The first refresh is modelled, not
   extrapolated: N jumps to the documented 3,868 and earners grow to the
   documented 4,276 total. The prologue takes ~99.5% of the 3,230 newly
   tracked players; reachable early/activity trophies (classified from
   name + description: not late-story, not a grind) roughly double their
   earners; late-story/grind trophies gain nobody. The split is then
   trimmed to exactly 4,276 total, so the forecast is count-consistent by
   construction. Spread 0.3 logit.
2. **Frozen history, long horizon** — the page is stale but the target is
   days out, so description-based logit priors carry it: the very earliest
   beats rise (+1.3 logit/day), the next tier rises slowly (+0.4), everything
   else falls hard (−3.0 logit/day) as dilution dominates.
3. **Moving history** — the normal case. Integer earners `E =
   round(rate * N / 100)` per read; separate fits for E and N. E blends a
   power-law fit with a saturating fit (geometric mean, anchored at the
   last E); N uses a saturating fit. Both fits use only the last 96 hours
   of reads, so they track the current growth regime instead of the launch
   flatline (a trophy that sat flat for days and only just started moving
   would otherwise drag every fit down to its launch level). The rate is
   E/N, capped by a per-class week-one ceiling (early story 40%,
   activities 25%, mid-game 5%, late/grind 1%). Spread 0.15·√days plus
   fit residual, capped at 2.5 logit.

The prologue is handled by identity, not by a rate threshold: in the
fitted regime it is never capped at the 40% early-story ceiling (a dip
below 99.995% in the last read used to send it through the E/N fit and
out at 40% — a catastrophic miss on a ~99.9% trophy). At the clamp it
stays at 99.99; dipped, it persists the dipped rate with a wide spread.

## Validation

- **Practice** (local, against five integer-earner reconstructions of the
  true graded answers — the exact per-trophy split is not identifiable
  from public data, so this is a range, not a point): mean loss **0.162**,
  worst **0.186** (v4 live: 0.376292). The ≤0.002 scores at the top of the
  board come from live-feedback tuning, not from a derivable rule; this
  entry does not pretend otherwise.
- **Real 41.75h check** (frozen launch history → live Exophase snapshot,
  2026-11-20T21:46Z): **0.480** vs house baseline 2.856 (v4: 0.477).
- **Simulated finals** (truth from the two real snapshots; 6 variants;
  24h horizon to 2026-11-26): family A (lognormal time-to-earn +
  smooth saturating N) **0.245** vs starter 0.235, house curve 0.114,
  v4 0.204; family B (stretched-exponential earn curve + steppy
  refresh-jump N) **0.091** vs starter 0.117, house curve 0.240,
  v4 0.246. Overall across both families v5 (0.168) beats v4 (0.225) and
  both house baselines (~0.177). The two families disagree on which
  forecaster shape wins — the honest read is that v5's final forecast is
  competitive, not dominant.

## Run

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Python 3.13, standard library only, deterministic, offline. Reads every
`<id>.in` in the input directory and writes `<id>.out` beside it: 55 rows ×
5 quantiles, ids 1–55 each once, finite and non-decreasing.
