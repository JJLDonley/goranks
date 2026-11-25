# Rank benchmark log (PLAN.md section 11)

Every experiment on the rank estimator and the rank model, in order, with its data, settings and result, so later
changes are measured against the same tables. Error is in ranks on the scale of `engine/rank_bench.h` (1k = 0,
1d = 1); "bias" > 0 means overestimated. Tool: `gifu --rank DIR --csv OUT.csv --out DIR`.

## R0-1 · 2026-11-23 · current estimator on 43 KGS 2015 dan games

- **Data:** u-go.net KGS 2015 (8,133 games, all with a 4d+ player). Even games, settled ranks, >= 120 moves,
  random sample stratified by Black's rank (up to 5 per rank, seed 7): 43 games, 86 player-games, 1d–9d.
  Files in `build/bench/kgs2015_s40/`, estimates saved as CGR[] in `build/bench/kgs2015_s40_cgr/`.
- **Engine:** KataGo OpenCL (Windows, Arc A770) with `b18c384nbt-humanv0`, JSON analyzer, 8 games in flight,
  ~14 moves/s for 12 profiles. About 10 minutes for 43 games (the GTP replay path would take ~100).
- **Estimator:** today's `rank_estimator.c`: 12 `rank_X` profiles (20k 15k 10k 7k 5k 3k 1k 1d 3d 5d 7d 9d),
  sum of log p(played move), floor 0.1 %, uniform prior; estimate = argmax and posterior mean.

| games | band | n | MAE argmax | MAE mean | bias | <= 1 | <= 2 |
|---|---|---|---|---|---|---|---|
| 1 | all | 86 | 2.56 | 2.54 | −2.43 | 14 % | 30 % |
| 1 | D1 | 3 | 1.00 | 0.83 | +0.22 | 67 % | 100 % |
| 1 | D2 | 5 | 1.40 | 1.09 | −1.09 | 40 % | 100 % |
| 1 | D3 | 5 | 0.80 | 1.09 | −0.75 | 60 % | 60 % |
| 1 | D4 | 5 | 2.20 | 1.80 | −0.94 | 20 % | 40 % |
| 1 | D5 | 5 | 2.00 | 2.00 | −2.00 | 0 % | 60 % |
| 1 | D6 | 8 | 2.75 | 2.54 | −2.54 | 0 % | 25 % |
| 1 | D7 | 23 | 2.70 | 2.82 | −2.82 | 13 % | 26 % |
| 1 | D8 | 15 | 2.87 | 2.79 | −2.79 | 0 % | 7 % |
| 1 | D9 | 17 | 3.41 | 3.46 | −3.46 | 6 % | 6 % |

(Runs of 5 games per player: only 2 samples, because the random sample has few repeated players.)

**Reading:** the estimate is compressed toward the middle dan ranks: low dans come out about right, and the
underestimate grows steadily with rank (9d is read as about 5.5d). Hypotheses to test, in order:
1. **Scale/era:** `rank_X` means KGS rank X in March 2020 (research report). KGS 2015 ranks may sit on a
   different scale. Check: the same benchmark on KGS 2018–2019 games.
2. **Coarse grid:** above 1d only 1d/3d/5d/7d/9d are queried. Check: query every rank 1d–9d.
3. **Ceiling:** the human net is documented as weaker at high dan, and a uniform prior over 20k–9d pulls toward
   the middle. Check: a calibration on top (R7) instead of the raw argmax.
4. **Probability floor (0.1 %):** a few surprising moves per game can pull strong players down. Check: floors
   of 0.01 % and 1 %.

## R0-2 · 2026-11-23 · offline variants on the R0-1 estimates (no engine)

`tools/rank_model/bench_offline.py build/bench/kgs2015_s40_cgr` re-scores the saved CGR[] (same moves, same
probabilities) with variants of today's estimator. Posterior mean vs declared, 86 player-games, 52 players.

| variant | MAE | RMSE | bias | <= 1 |
|---|---|---|---|---|
| floor 0.01 %, all moves | 2.57 | 2.89 | −2.47 | 15 % |
| **floor 0.1 % (today), all moves** | 2.54 | 2.85 | −2.43 | 14 % |
| floor 1 %, all moves | 2.39 | 2.72 | −2.28 | 17 % |
| floor 0.1 %, skip first 20 moves | 2.23 | 2.58 | −2.08 | 23 % |
| floor 1 %, skip first 20 moves | 2.06 | 2.43 | −1.86 | 27 % |
| linear calibration, leave-one-player-out | **1.22** | 1.55 | +0.01 | 50 % |
| baseline: always guess the others' mean | 1.89 | 2.33 | −0.01 | 36 % |

**Findings:**
- The floor matters little (hypothesis 4 mostly rejected): 1 % is slightly better than 0.1 %.
- Skipping the opening (first 20 of a player's moves) helps by ~0.3 ranks: memorised openings say little about
  strength, and they are where the AI-era / pre-AI difference is largest.
- The calibration slope on all data is **0.96**: the estimator already orders and spreads these dan players like
  their declared ranks; the error is almost all a **constant offset (~ −2.4 ranks)**. A calibration that removes it
  beats the guess-the-mean baseline clearly (1.22 vs 1.89), so it is not just shrinkage on a narrow range.
- This favours hypothesis 1 (KGS 2015 dan ranks sit ~2 ranks above the net's KGS-2020 ranks, or the net
  underrates strong players uniformly) over compression. Next: the same design on KGS 2018 (closer to 2020) and on
  OGS kyu games, to see whether the offset depends on era and rank.

## KGS bot screen (index_games.py, all of u-go.net 2000 – 2019-04, 228,849 games)

Accounts with >= 300 games, by games per active day: the engines (Zen19K 46, Golois 26–32, Grant10 26, ZenLv8 25,
Zen19L/X 22, ZenLv9 19, petgo3 18, yagumo 17, MIss 16, daien 14, Erinys 13) are separated from the busiest humans
(<= 8) by a clear gap, so the rule is >= 300 games at >= 12 per active day (`f_busy`), listed for review in
`<index>_busy_accounts.txt`. With the name list, 116,739 games are clean. Clean games per year fall from ~8–12k
(2004–2014) to 2.6k (2017), 1.7k (2018) and 0.4k (2019-01..04): those years are dominated by bot games.

## Engine throughput: fixed groups vs sliding window (2026-11-23)

`--rank` on the JSON analyzer first sent games in fixed groups of 8 and waited for each group's slowest game; on
the 166-game run that gave ~1.3 games/min (~6 min per group of 8). Keeping 8 games in flight and refilling a slot
as soon as its game finishes (`rank_games_batched` in `app/headless.c`) gives ~3.5 games/min on the same games
and GPU (Arc A770, OpenCL): about 2.6x, with the same RAM bound (8 games in flight).

## R0-3 · 2026-11-23 · 12 KGS 2015 players x 15 games (6d–9d)

- **Data:** clean KGS 2015 games of players with >= 15 clean games and a stable rank (max − min <= 1 over the 15),
  up to 3 players per rank: only 6d–9d qualify (the archive holds games with a 4d+ player). 166 games,
  332 player-games, 63 players (opponents included). `build/bench/kgs2015_p15/`. 30 min on the sliding window.

| games | band | n | MAE mean | RMSE | bias | <= 1 | <= 2 |
|---|---|---|---|---|---|---|---|
| 1 | all | 332 | 2.84 | 3.05 | −2.81 | 7 % | 23 % |
| 1 | D6 | 108 | 2.65 | 2.84 | −2.62 | 9 % | 24 % |
| 1 | D7 | 67 | 2.47 | 2.67 | −2.42 | 4 % | 37 % |
| 1 | D8 | 93 | 3.26 | 3.45 | −3.22 | 3 % | 15 % |
| 1 | D9 | 60 | 3.08 | 3.25 | −3.08 | 5 % | 13 % |
| 5 | all | 48 | 2.96 | 3.03 | −2.96 | 0 % | 6 % |
| 15 | all | 12 | 3.04 | 3.11 | −3.04 | 0 % | 8 % |

Offline (`bench_offline.py`): skip 20 + floor 1 %: MAE 2.33; linear calibration (LOPO) 0.81 vs guess-the-mean 1.06
(slope 0.48 on this narrow 6–9d range).

Per player, all games pooled (skip 20, floor 0.1 %):

| declared | estimates |
|---|---|
| 6d (Dom, ohio, swedish1k, waage) | 3.0, 3.0, 3.0, 3.3 |
| 7d (Pan8, goliard, roln111) | 4.9, 5.0, 5.0 |
| 8d (misterious, KingJulien, Allosaurus, Crazywind, Joker7) | 4.6, 5.0, 5.0, 5.0, 6.6 |
| 9d (cloudnotes, machine, kingmaker) | 5.0, 5.0, 5.0 |

Mean log-likelihood per move of each dan profile (skip 20, floor 0.1 %):

| players | 1d | 3d | 5d | 7d | 9d |
|---|---|---|---|---|---|
| 6d | −1.284 | **−1.233** | −1.245 | −1.278 | −1.376 |
| 9d | −1.400 | −1.286 | −1.234 | **−1.224** | −1.260 |

**Findings:**
- **More games do not help** (−2.8 at 1 game, −3.0 at 15): the error is systematic, not noise.
- **Grid lock (hypothesis 2 confirmed):** with many moves the posterior is so peaked that the estimate snaps to a
  queried profile (3d or 5d). The estimator must query every rank, or interpolate between profiles.
- **Ceiling (hypothesis 3 confirmed):** on 9d players' moves the 7d profile fits better than the 9d profile, and
  7d vs 5d differ by only 0.010 nats per move. Above ~5–7d the human net does not separate ranks, so the plan's
  two regimes are needed: engine-loss / net-size features must carry the estimate at high dan (R3, R7).
- 6d KGS-2015 players read as 3d: a ~3-rank offset at 6d (hypothesis 1), compared next with KGS 2018 and OGS.
- The account "machine" (8.9d) scores like the other 9ds, nothing bot-like in the rank fit; kept.

## R0-4 · 2026-11-23 · current estimator on 49 KGS 2018 – 2019-04 games (1d–9d + pro)

Same design as R0-1 (clean even games, >= 120 moves, up to 5 per Black rank, seed 11), from the years closest to
the human net's rank date (KGS, 2020-03-01). `build/bench/kgs2018_s40/`, 98 player-games.

| games | band | n | MAE mean | RMSE | bias | <= 1 | <= 2 | band | band ±1 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | all | 98 | 1.76 | 2.42 | −0.84 | 48 % | 66 % | 26 % | 58 % |
| 1 | D1 | 5 | 0.57 | 0.70 | −0.33 | 80 % | 100 % | 60 % | 100 % |
| 1 | D2 | 7 | 1.01 | 1.09 | −0.70 | 43 % | 100 % | 14 % | 86 % |
| 1 | D3 | 5 | 1.47 | 1.98 | +0.86 | 40 % | 80 % | 40 % | 60 % |
| 1 | D4 | 7 | 1.70 | 2.32 | +0.17 | 43 % | 71 % | 29 % | 71 % |
| 1 | D5 | 5 | 0.94 | 1.80 | +0.73 | 80 % | 80 % | 80 % | 80 % |
| 1 | D6 | 8 | 1.55 | 1.85 | +0.02 | 50 % | 62 % | 12 % | 50 % |
| 1 | D7 | 19 | 1.86 | 2.26 | −1.02 | 32 % | 68 % | 21 % | 42 % |
| 1 | D8 | 22 | 1.73 | 2.20 | −0.59 | 64 % | 68 % | 5 % | 68 % |
| 1 | D9 | 14 | 1.85 | 2.97 | −1.85 | 50 % | 50 % | 50 % | 50 % |
| 1 | Pro | 6 | 4.46 | 4.70 | −4.46 | 0 % | 0 % | 0 % | 0 % |

**Finding: the KGS rank scale moved between 2015 and 2018–19 (hypothesis 1 confirmed).** Same estimator, same
net: the overall bias falls from −2.4 (2015) to −0.8 (2018–19); at 6d from −2.5 to 0.0, at 8d from −2.8 to −0.6.
A KGS 2015 dan rank meant ~2–3 ranks more strength than the same label in 2018–20 (KGS ranks deflated when strong
bots and AI-trained players arrived; the net's `rank_X` is the 2020 scale). So the offset is era-dependent, and R1's
per-(server, period) offsets are required: a single global calibration would be wrong for every era but one.
Pros are read ~4.5 ranks low (the net's ceiling: `rank_9d` is the top), and the 9d band still shows −1.85.

## R0-5 · 2026-11-23 · current estimator on 126 OGS games, 2019–2021, all bands

OGS 2021 dump sample (`sample-100k.json.gz`), converted by `ogs_index.py` with each player's rank at game time;
clean even games, >= 120 moves, 2019+, up to 12 per band of the two players' mean rank. `build/bench/ogs2019_s/`,
252 player-games.

| games | band | n | MAE mean | RMSE | bias | <= 1 | <= 2 | band | band ±1 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | all | 252 | 2.72 | 3.97 | +1.18 | 32 % | 57 % | 25 % | 60 % |
| 1 | K1 | 21 | 6.80 | 8.73 | +6.42 | 10 % | 29 % | 33 % | 48 % |
| 1 | K2 | 28 | 4.92 | 5.85 | +4.24 | 11 % | 21 % | 18 % | 57 % |
| 1 | K3 | 22 | 3.35 | 4.28 | +2.02 | 27 % | 41 % | 41 % | 64 % |
| 1 | K4 | 26 | 1.56 | 1.94 | +0.82 | 42 % | 69 % | 42 % | 96 % |
| 1 | K5 | 28 | 1.49 | 1.98 | +0.42 | 46 % | 79 % | 50 % | 82 % |
| 1 | D1 | 27 | 1.37 | 1.98 | −0.28 | 59 % | 78 % | 41 % | 70 % |
| 1 | D2 | 28 | 1.70 | 2.39 | +0.07 | 46 % | 79 % | 7 % | 68 % |
| 1 | D3 | 21 | 1.87 | 2.25 | +0.09 | 24 % | 71 % | 0 % | 48 % |
| 1 | D4 | 20 | 2.10 | 2.46 | −0.32 | 25 % | 55 % | 0 % | 40 % |
| 1 | D5 | 10 | 2.67 | 3.06 | −0.20 | 20 % | 40 % | 20 % | 20 % |
| 1 | D6–D9 | 21 | — | — | −0.1 … −3.3 | small n | | | |

Median single-game estimate by declared rank: 15k → 7.1k, 13k → 5k, 10k → 6.9k, 8k → 6.9k, 6k → 4.4k, 4k → 3.3k,
3k → 3k, 1k → 0.4d (≈ 1d), 1d → 1.1d, 3d → 3.6d, 5d → 3.9d, 7d → 8.8d. Only 1 of 252 estimates sits at the 20k floor.

**Findings:**
- **OGS 1d–4d agree with the net's scale** (bias −0.3 … +0.1): OGS dan ranks of 2019–21 ≈ KGS 2020 dan ranks.
- **Kyu compression:** single-game estimates of 8k–15k players all land around 5k–7k (slope ≈ 0.3 over the kyu
  range). Not the 20k floor; weak players' moves look like mid-kyu moves to the net, and the queried kyu profiles are
  sparse (20k, 15k, 10k, 7k, 5k). A fixed offset cannot fix this: it needs the finer grid and a learned model on the
  full feature set (R3 + R7), and kyu bands need more than one game.

## R0 summary (the baseline every later model must beat)

| corpus | player-games | MAE | bias | <= 1 rank | main error |
|---|---|---|---|---|---|
| KGS 2015 dan (R0-1) | 86 | 2.54 | −2.43 | 14 % | era offset (2015 ranks ≈ 2–3 stronger) |
| KGS 2015 6–9d, 15 games/player (R0-3) | 332 | 2.84 | −2.81 | 7 % | offset + grid lock + ceiling |
| KGS 2018–19 dan (R0-4) | 98 | 1.76 | −0.84 | 48 % | high-dan ceiling, pros |
| OGS 2019–21 all bands (R0-5) | 252 | 2.72 | +1.18 | 32 % | kyu compression |

## R3 feature extraction cost (2026-11-23, 16 KGS games, 3,438 moves, 29 rank profiles, Arc A770 OpenCL)

| KataGo analysis settings | engine visits | time |
|---|---|---|
| 12 analysis threads x 2 search threads (Gifu default) | 50 | 1219 s |
| 48 x 1, default nnMaxBatchSize | 50 | KataGo exited after 67 s (no error text captured) |
| 64 x 1, nnMaxBatchSize 64 | 50 | 967 s |
| 64 x 1, nnMaxBatchSize 64 | 0 (human policy only) | 467 s |
| 64 x 1, nnMaxBatchSize 64 | 8 | 559 s |

- Many 1-visit queries batch better with many analysis threads of 1 search thread (KataGo's own advice), but
  only ~21 % here; raising the thread count needs nnMaxBatchSize raised with it (48 x 1 without it crashed).
- The engine search was the larger cost at 50 visits. **8 visits are enough:** point loss at 8 vs 50 visits
  correlates 0.91 per move and **0.963 per player-game** (mean clipped loss, the model's feature); mean loss 1.14
  vs 1.39 (shallower search finds slightly smaller losses). `--rank-features` now defaults to 8 visits.
- With 1-visit queries the human-policy values of the same move differ between runs by a median 0.010 (KataGo
  picks a random board symmetry per query): noise the per-game averages absorb.
- Profile grid: on these (dan) games, today's 12 profiles move the grid estimate by 0.30 ranks on average (max 1.22)
  vs all 29; thinning only the kyu profiles changes nothing here, but these games have no kyu players, so all 29 are
  kept until OGS kyu data says otherwise.

## R7-1 · 2026-11-23 · first trained models, 280 KGS games 2006–2019 (560 player-games, 386 players)

Features from `--rank-features` (29 `rank_` profiles, 8 visits, all moves), `train_rank.py`, 5-fold
cross-validation **grouped by player**, moves 1–40 skipped. Declared ranks are KGS ranks of their year.

| model | MAE | RMSE | bias | <= 1 |
|---|---|---|---|---|
| A likelihood over all 29 profiles, no training | 2.63 | 3.26 | −2.28 | 19 % |
| B A + linear calibration | 1.59 | 2.05 | 0.00 | 40 % |
| C ridge, human features (λ 100) | 1.54 | 1.95 | −0.01 | 43 % |
| D + engine features (λ 100) | 1.53 | 1.95 | −0.01 | 40 % |
| E + server/period offsets (λ 100) | **1.51** | 1.92 | −0.01 | 41 % |

A by era: pre-AI bias −2.68, early-AI −1.60, current −0.39 (the era offset of R0-4 again). E by band: D1 +3.23,
D2 +2.71, D3 +1.52, D4 +1.55, D5 +0.68, D6 +0.18, D7 −0.41, D8 −1.09, D9 −1.59, pros −2.00.

**Findings:**
- Learning on the features cuts the error from 2.63 to 1.51 ranks per single game.
- **Regression to the mean:** the set is 70 % 7d–9d (the KGS archive's mix), so the model pulls everyone toward
  ~7d: low dans read 2–3 ranks high, 9d and pros 1.6–2 low. The fix is a balanced training set (the stage-1 design:
  100 player-games per rank per era), not a better regressor.
- Engine features add little on dan-only single games (1.54 → 1.53); their value should show at high dan once the
  set is balanced, and with several games per player.
- Era offsets help a little even within KGS (1.53 → 1.51).

## Stage-1 design (2026-11-23)

The user asked for ~1000 player-games per rank per era. Available data covers it (101 of 102 cells of 25k–9d x
pre/early/current have >= 1000 clean player-games; 9d 2016–18 has ~505), but the cost is ~44,000 games, ~7–8 days
of GPU even with move sampling. Staged instead: **stage 1 = 100 player-games per (rank 20k–9d, era)**, ~2,900 cells'
worth, mixing KGS and OGS half-half where both exist, at most 3 games per player per cell, moves 1–40 skipped and
30 moves sampled per player, 8 visits, 64x1 analysis threads with nnMaxBatchSize 64 (`tools/rank_model/run_stage.sh`).
Later stages grow the cells where the model's error still falls with more data.

OGS bot screen, first version, counted correspondence games too and flagged 6,948 accounts at >= 12 games per
active day: mostly humans with dozens of simultaneous correspondence games. Fixed: the volume is counted on live and
blitz games only (`ogs_index.py`, `reflag_busy.py`).

## Bot exclusion, tightened (2026-11-23)

Bot games corrupt both the rank labels and the AI-usage baseline, so they are excluded as strictly as the data
allows, in three layers:
1. **Names:** the KGS / OGS engine accounts known from 2005–2025 (exact names, e.g. AQ, abakus, PhoenixGo,
   darkforest, DCNN, NeuralZ, AyaMC, Golois, Zen*, petgo3, GnuGo, Leela*, KataGo*, amybot*) plus patterns
   (`bot`, `^zen\d`, `^golois`, `^leelaz`, ...). `index_games.py`.
2. **Volume:** >= 300 games at >= 12 per active day (live / blitz only on OGS). `f_busy`.
3. **OGS live API:** every OGS account in a selected training set is looked up once (`ui_class` contains "bot" for
   bot accounts; checked: amybot-ddk -> "bot"), at <= 1 request/s, cached in `~/gorank-data/ogs/player_class_cache.tsv`;
   games with a bot are moved out before features are extracted (`ogs_bot_check.py`, step of `run_stage.sh`).
KGS has no such API: there layers 1–2 apply, and the AI-usage model (R5) will later flag what still looks
engine-made. Re-index after the longer name list: 116,698 clean KGS games (−41: most of those accounts' games were
already out as busy or handicap).
- OGS volume rule, second fix: counting live / blitz only still flagged 6,804 accounts, mostly human-looking names at
  13–19 games per active day: OGS humans play many quick 9x9 / 13x13 games, while the 12/day threshold was found on
  KGS's 19x19-only archive. Next index / re-flag counts 19x19 live / blitz games only. Stage 1 runs on the v2 flags:
  the extra false positives cost some human games but let no bot in (the API check still runs on every account).

## Stage 1 selection (2026-11-23)

`select_stage.py 100`: 5,528 games (906 KGS, 4,622 OGS) for 8,699 player-games; every (rank 20k–9d, era) cell has
100 except current-era 9d (99). 6,000 kyu and 2,699 dan player-games. Kyu cells are OGS in every era (KGS bulk has
almost no kyu games); 1d–6d about half KGS / half OGS pre-AI and early-AI; 7d–9d pre-AI and early-AI KGS only;
current-era dans mostly OGS. 19x19 only (non-19x19 games are never clean; `--rank-features` also skips them).
Engine use: KataGo's JSON analysis engine (batch mode): one query per game per profile covering all its sampled
turns, plus one engine query; 8 games in flight; 64 analysis threads x 1 search thread, nnMaxBatchSize 64.

## Move to GoRanks, standalone extractor (2026-11-23)

Training and evaluation moved out of Gifu's C code into `Gifu/GoRanks` (Python, not part of the executable).
`gorank/features.py` talks to KataGo's JSON analysis engine directly. Two things it needed, found by comparing with
Gifu's old C extractor on the same moves:
- **Binary, unbuffered pipes** to the Windows KataGo started from WSL: with text-mode line buffering KataGo never
  received the queries (it sat idle); one query in binary mode answers in 7 s.
- **`ignorePreRootHistory: false`** in every query (Gifu sent it): without the move history the human net does not
  know the last move, and forced replies drop from ~0.87 to ~0.21. With it, per-move probabilities agree with the C
  extractor to a median 0.017 (max 0.105), i.e. KataGo's random-symmetry noise; point loss to a median 0.39.
Benchmark runs and their SGFs were copied to `~/gorank-data/runs/bench/`.

Reference method (from go-performance-quality, CC BY-NC-SA 4.0; methods only, no code copied): an Elo ladder for the
human profiles (from a tournament between them), a Position Elo per position and loss threshold (0.05 ... 24
points: the Elo at which a profile's policy has even odds of a move under the threshold), and a Bradley–Terry player
rating with errors clustered by position. Candidate next model after the stage-1 baseline: it adjusts for position
difficulty, which is what grid lock and kyu compression call for. It needs per-profile policy mass under each loss
threshold and engine losses for the moves humans would play (more visits / candidates).
- **Pacing:** the first stage-1 run stalled the same way at full size (8 games x 30 queries, ~0.8 MB written while
  KataGo was still loading its networks: Python blocked in the pipe write, KataGo idle). Fix: wait for KataGo's
  "ready" line, then keep at most 40 queries outstanding and send the next as one completes. 16 stage games (skip 40,
  sample 30, 29 profiles, 8 visits): 241 s, ~4 games/min, so stage 1 (5,528 games) is ~23 h of GPU; the model is
  trained on the partial CSVs as they grow.

## Ongoing training in rounds (2026-11-23, the user's idea)

`gorank/rounds.py`: each round adds 5 player-games per (rank 20k–9d, era) cell (cumulative target 5 x round, so
cells that could not fill catch up later), extracts their features (one KataGo), retrains on everything so far
(`train_rank.py`, CV grouped by player) and appends a row to `reports/learning_curve.csv` (overall MAE / RMSE /
bias / within ±1 and MAE per band and era). The curve shows where more data still helps, so later stages grow only
those cells. Resumable (`rounds/state.json`).

Filtering before a game can enter a round: everything in the index flags (19x19, even, settled ranks, >= 100 moves,
no timeout / cancel / early resignation, ranked, no correspondence, bot names, volume rule) and, for OGS, both
accounts cleared by the live API check. The API check stays at 1 request/s (no bulk endpoint: `id__in` filters are
rejected, `ui/bot/list` does not exist, and neither OGS dump carries a bot marker); it runs in random account order
so early rounds are not skewed toward old (pre-AI) accounts. At the start of round 1, 139 OGS games were cleared,
so round 1 filled 49 of 87 cells (mostly dan cells from KGS); kyu cells fill as the check proceeds.

### Round 1 (2026-11-23)
178 games selected, 234 player-games trained (KGS dans + the first cleared OGS games): **MAE E 2.04 vs A 2.35**,
32 % within ±1 rank, 30.5 min. KataGo died silently after 25 of the round's 84 OGS games (its log just stops: no
error), so 59 OGS games were not extracted. Fixes:
- `features.py` restarts KataGo when it dies and goes on with the games not yet started; the games in flight at
  the crash are retried one at a time afterwards, and one that kills KataGo on its own is quarantined
  (`<out>.quarantine.txt`) for inspection.
- `rounds.py` rebuilds "done" from what was actually extracted (rows in the feature CSVs, quarantine lists, rounds
  whose extraction finished normally) and recounts cells from it, so games lost to a crash go back to the pool.

### OGS bot check, stage 1 (finished 2026-11-23 ~19:20)
All 6,161 OGS accounts in stage 1 classified (0 unknown); **94 of 4,622 OGS games (2 %) had a bot** and were moved
to `sgf/bot_games` (none had entered a round: rounds only take games whose accounts were already cleared). The bots
include obvious names (KataGo / Leela / Golaxy variants) and human-looking ones (Mortaloid, Spayde, Tom_grc, Pablo
LION, Zero Go, idontca1): the name list alone would have missed most, so the API check stays mandatory for OGS. The
28 accounts are now also in `index_games.py`'s name list.

### Round 2 (2026-11-23 ~20:00): first kyu data
342 games, 854 player-games (rounds 1+2; 524 kyu, 330 dan): **MAE E 2.51 vs A 3.04**, 25 % within ±1. Higher than
round 1 (2.04) because the set now has kyu players, who are harder (R0's kyu compression), not a regression: by band
E beats A at the extremes (K1 2.75 vs 3.82, K3 2.11 vs 3.41, D7 1.58 vs 2.76) but is worse mid-range (D1 3.01 vs
2.13), and the errors still regress to the middle (9d −3.05, 6d −2.14, K1 +1.75). Round 2 OGS extraction: 282 games,
0 failures, 0 KataGo restarts (the crash of round 1 did not recur).

Experiments on rounds 1+2 and on the 280 all-move KGS games:
- **Model F** (E + piecewise-linear calibration with bends at 15k 10k 5k 1k 3d 6d): MAE 2.51, no better than E. The
  mapping is not the problem.
- **Move sampling** (30 per player vs all ~100), same 280 KGS games: E/F MAE 1.60 vs 1.49. Sampling costs ~0.1 rank
  for a 2–3x speed-up: kept.
- So the remaining error is mostly single-game noise (plus label noise: OGS kyu ranks, K1 spanning 20k–13k), which
  any regression turns into regression to the mean. Next: more data (rounds), per-player multi-game estimates, and
  a richer per-move signal (Position Elo: engine loss over the moves humans consider, not just the played move's
  probability).

### First usable model (2026-11-23)
`train_rank.py ... --out ~/gorank-data/models/rank_r002.json` (rounds 1+2, best model by CV MAE) and
`gorank/estimate.py MODEL GAME.sgf [--server ogs|kgs]` (extracts with KataGo, all moves by default; `--features CSV
--file NAME` reuses extracted rows). Example (training games, so in-sample — a demo, not a test): 20k -> 18k,
18k -> 18k, 14k -> 8k, 13k -> 9k, 1d -> 2k, 1d -> 3d (likelihood-only: 15k, 18k, 5k, 5k, 1k, 7d). The model pulls the
raw likelihood toward the truth, but SDK–DDK games still read too strong (the kyu compression).

## Game phases: overall, opening, middle, end (2026-11-23, the user's request)

Four estimates per player: **overall** (every move after 40, as before, for comparability), **opening** (moves
1–60), **middle** (61–150), **end** (151–). From round 4 on, extraction samples 10 moves per player per phase
(`features.py --per-phase 10`; the opening is included), the same ~30 moves per player as before, so no extra GPU.
Each round trains one model per phase (`train_rank.py --phase`, min 8 moves per player-game for a phase), exports
`~/gorank-data/models/rNNN_<phase>.json` and `latest_<phase>.json`, and writes one learning-curve row per phase.
`estimate.py ~/gorank-data/models GAME.sgf` prints all four. On rounds 1+2 (old sampling, moves 41+ only):
overall 2.51 (854), opening 3.09 (82: moves 41–60 only), middle 2.81 (854), end 3.30 (539).

## Every move, raw analysis saved, 17 profiles (2026-11-23, after the user's review)

The user pointed out two flaws: (1) only derived features were saved, so any new feature would need KataGo again;
(2) sampling 10 moves per phase does not measure a phase. Changes:
- **Every move of every game** is analysed (no sampling, no skip); phases come from move numbers.
- **Raw analysis saved** per game (`features.py --raw`, `rounds_full/raw_rNNN_<server>.jsonl.gz`, ~0.6 MB/game):
  top-24 human-policy moves per turn and profile (+ the played move), and every engine candidate (move, order,
  visits, winrate, scoreLead, scoreStdev, prior, lcb, utility) plus the root. Each game sees KataGo once; later
  features and models are computed from these files.
- **17 profiles** instead of 29: every dan rank (1d–9d) and kyu ~3 apart (20k 17k 14k 11k 8k 5k 3k 1k). Each profile is
  one network evaluation per move (the profile is an input to the human net), so this is ~40 % less GPU per move.
- New rounds directory `rounds_full` (the sampled rounds 1–3 stay in `rounds/` as a legacy comparison).

**Where the time goes (measured):** per move = 29 human-net evaluations (one per profile) + 8 engine visits ≈ 37
evaluations; ~250-move game ≈ 9,000. The pipeline reached ~230 evaluations/s. `katago benchmark` on the Arc A770
(OpenCL, b18c384nbt): ~225 visits/s at 16 threads (batch ~8), **~253 nnEvals/s at 64 threads (batch ~41)** — so the
pipeline already runs at ~90 % of the card's ceiling; the GPU is the limit, not batching or Python. 2 full games
(422 moves, 29 profiles): 118 s. Faster options: fewer evaluations per move (done: 17 profiles) or a faster GPU
(NVIDIA + TensorRT; to be benchmarked on a rented card before committing to it).

## R5 first pass: engine-likeness on clean games (2026-11-23)

`gorank/ai_scores.py` scores each player-game from the saved raw analysis: e(m) from KataGo's candidates weighted by
point loss (temperature 1, floor 5 %), h(m) from the human net at the declared rank (nearest saved profile), weight
from the human-expected loss over the engine's candidates, then the two-state HMM (enter 0.02, leave 0.1). Moves
1–40 and decided positions left out. On the first 51 full-move KGS player-games (round 1): log Bayes factor median
−0.34, 90 % 0.37, max 7.38; 6 player-games have a stretch with P(assisted) > 0.5. **All high scores are 7d–9d, and
2 of the 4 highest are from 2004–2005**, before any superhuman engine existed: false positives from strong players
whose moves are engine-like, made worse by the human net's ceiling (`rank_9d` underrates real 9d play). Median log
BF by band: 1–4d −0.39, 5–7d −0.41, 8–9d −0.26. Next (R6): thresholds per band from each band's own clean-game null
(pre-2016 games are guaranteed clean controls), and a stronger human reference for 7d+ (e.g. `proyear_2023`) instead
of the saturated `rank_9d`. Nothing is shown to users from this until calibrated.

## Engine-likeness detail, ownership, groups (2026-11-23, the user's direction)

One layered analysis, nothing hidden behind summaries: `ai_scores.py OUT.csv RAW... --moves MOVES.csv` writes the
per-player-game summary and **one row per analysed move** (phase, played move, engine best and the played move's rank
among the candidates, point loss, human / engine probability, human-expected loss, critical flag, weight, llr,
P(assisted)). The played move's loss now comes from consecutive roots (score before − score after), so every move
has one (2,912 of 2,912; before, moves outside the 8-visit candidates had none).

Round 1 (85 player-games, every move) by band — found the engine's best on all moves / on critical moves (a human of
that rank would lose >= 2 points on average there):

| band | n | top-1 all | critical / game | top-1 critical | log BF median | stretch P > 0.5 |
|---|---|---|---|---|---|---|
| <= 13k | 15 | 21 % | 3.9 | 1 % | −0.49 | 1 |
| 12–7k | 9 | 34 % | 5.7 | 21 % | −0.50 | 0 |
| 1–4d | 17 | 46 % | 2.0 | 36 % | −0.40 | 0 |
| 5–7d | 17 | 50 % | 4.9 | 26 % | −0.41 | 3 |
| 8–9d | 22 | 57 % | 2.0 | 28 % | −0.26 | 4 |

Engine agreement rises with strength; finding the best move at critical moments separates weak players most (1 %
vs ~30 % for dans). Flagged stretches are all 5d+ (the high-dan false positives).

Next layers (added as columns to the same per-move rows):
- **Ownership** is now requested with the engine query (`includeOwnership`, same network pass, no extra GPU) and
  saved per turn in the raw file (−100..100, Black's side) from full-move round 2 on.
- **Groups:** `gorank/board.py` replays a game (captures, suicide), lists groups with liberties, and reads a group's
  status from ownership (alive >= 0.6, dead <= −0.6, else unsettled; tested). Planned events: a group of the player's
  (or the opponent's) unsettled before the move, and after it saved / killed or lost — "found it" vs "missed it".
- **Consistency:** per player, how often they find moves of a given difficulty for their rank (h of the right move)
  across earlier moves and other games; a run of finds far above the player's own rate vs a steady habit.

## Layered report (2026-11-23, the user's structure)
All results are reported as **rank band -> individual rank -> per game (player)**, each level split **overall /
opening (1–60) / middle (61–150) / end (151–)**: `report.py MOVES.csv PREFIX` -> `PREFIX.csv` (one row per level,
band, rank, game, phase) and `PREFIX.html` (nested, expandable). Every level aggregates the per-move rows below it.
The per-move table now keeps every move (moves 1–40 too); only the AI-use evidence (llr, P(assisted)) leaves out 1–40,
marked by `counted`. Round 1, band level, overall / opening / middle / end — mean point loss per move:
K1 4.05 / 2.46 / 5.25 / 7.90 (top-1 18 %), D7–9 0.79 / 0.40 / 1.17 / 1.03 (top-1 51 %). Rank estimates per phase
and group events will be columns in the same structure.

## Difficulty, first pass (2026-11-23, the user's request after Zhangqi's top-amateur difficulty)

Per position (`ai_scores.py`, per-move columns): P(a player of rank r loses >= 2 points), r = 1d / 5d / 7d / 9d, from the
saved human policies over the moves with a known loss (engine candidates, candidate searches, the played move from
consecutive roots); `coverage_7d` = the 7d profile's probability mass with a known loss; `difficulty_rank` = the rank
with even odds of not erring (interpolated across the 17 profiles). Rolled up in the layered report per band / rank /
game and phase: mean P(7d errs), hard positions (P(7d errs) >= 0.5) faced and solved (player lost < 2), and the
90th-percentile difficulty rank as the game's difficulty rating.

Round 1, 6,995 moves: 7d coverage median 0.82 (p25 0.60) even without candidate searches; mean P(7d errs) 0.10;
660 positions with P(7d errs) >= 0.5; 82 % of positions are easy for every rank, ~7 % "harder than 9d" (suspicious:
8-visit losses are noisy in wild positions).

| band | player-games | P(7d errs) | hard | solved | difficulty p90 |
|---|---|---|---|---|---|
| K1 | 24 | 0.17 | 168 | 14 | pro |
| K5 | 19 | 0.10 | 92 | 31 | 3d |
| D1–3 | 24 | 0.07 | 79 | 27 | 6k |
| D7–9 | 35 | 0.08 | 130 | 56 | 14k |

Weak players' games rate *harder*: partly real (they create wild positions where only one move holds), partly noise
(8-visit losses in chaotic positions). Solved-share behaves as expected (K1 8 %, D7–9 43 %). Before the rating is
trusted: calibrate P(error) against real 7–9d error rates (KGS), and improve losses with the candidate searches
(`features.py --candidates 3`, built, off until tested) and deeper searches on critical positions (R3c visit ladder).
- **Reference set to 9d (the user's rule):** difficulty = P(a 9d loses >= 2 points); hard = P(9d errs) >= 50 %; game
  difficulty rating = mean P(9d errs) (%); candidate searches use the 9d profile's likely moves. Round 1 by band:
  K1 13.4 % (hard 132, solved 15), K5 8.8 % (80 / 27), D1–3 6.7 % (67 / 26), D7–9 6.9 % (129 / 56).

## Report naming and rates (2026-11-23, set by the user)
Phases Overall / Opening / Middle / Endgame; columns Games Analyzed, Moves Analyzed, Best Move %, Error Rate %, Key Moment
Top Move %, Game Difficulty (9d) %, Tough Spots, Key Move Found %, AI-like moves; individual games add the AI-like span
(longest run of the player's AI-like moves, e.g. "87–123 (19 moves)"). Error rates instead of per-move scores: Error
Rate % (player's moves losing 2+ points) next to Game Difficulty (9d) % (a 9d's error rate on the same positions).
Round 1: K1 50.5 % vs 13.3 %, K5 22.4 % vs 8.7 %, D1–3 13.5 % vs 6.7 %, D7–9 10.3 % vs 6.9 %. The same KGS 2019 game
shows AI-like spans for both players over the same moves (87–123 / 90–116): a forcing sequence, not engine use — the
calibration must discount forced sequences.

## Viewer (2026-11-23, the user's idea)
`viewer/index.html`: a single static page (no server, works from disk) — Overview (the Rank Estimate Training table
and charts by rank band), Ranks (the same by rank), Games (filterable list; each game opens its Overall / Opening /
Middle / Endgame table with the AI-like span, per-move graphs of point loss, Game Difficulty (9d) with Tough Spots, and
Engine Likeness with the AI-like spans shaded, and the move list). Hover tooltips on every chart. Data from
`gorank/viewer_export.py` (`viewer/data/summary.js` + one file per game, loaded on demand), refreshed after every
round by `rounds.py`. Example lead found with it: OGS 1k game 69276012 (Black), AI-like span moves 43–79, 7 of 7 Tough
Spots solved in that span — to inspect once calibrated.

## Viewer: one game = one entry, board, handicap (2026-11-23, the user's requests)
Games are no longer split by player: the Games list has one row per game (Black and White side by side, date,
handicap), and a game opens one view — both players in one Overall / Opening / Middle / Endgame table, one board, and
per-move graphs with both players (click any bar / point / move row to show that move on the board; arrow keys
step; a cursor line marks the current move). The board panel (left, sticky, Gifu-like but secondary) shows the
position before the move with the played move and the AI's best marked, and the move's loss, 9d error chance, Key
Moment, Tough Spot and Engine Likeness. Every game is marked Even / Handicap N (stage 1 only selects even games).
Band / rank totals still count each player in their own rank; partially analysed games are shown but left out of
totals. The per-game table images / CSV list each game's two players together.

## Decided-position rule replaced; bot purge; partial games out (2026-11-23, after the user's questions)
- **"Decided" rule**: borrowed from the research (Regan: errors in decided positions say little) as KataGo win rate
  < 5 % or > 95 %. Measured on 50,367 analysed moves it excluded **66 %** of all moves (KataGo's win rate saturates
  early in amateur games; 29 % of moves were "decided" with the score within 20 points). Score-based alternatives:
  lead > 10 / 15 / 20 / 30 points excludes 57 / 46 / 38 / 26 %. Now: **every move counts** in the rank and difficulty
  totals; a lead over 30 points marks a position decided for the **Engine Likeness evidence only** (shown greyed per
  move). With more moves as evidence, 54 of 470 player-games now show a stretch with P(assisted) > 0.5 — still
  uncalibrated.
- **Bot purge**: a KGS name scan found 2016–2019 engine accounts the rules missed (ELF / ELFv2 / ELFv2v2 / Elfv2Blitz,
  Leela Zero nets LZ100 / LZ202 / LZ203, DeepGenZo, Saireborn, DCNN, Royal*, AlphaGo imitators, ...). 44 stage-1 KGS
  games had one; `gorank/purge_games.py` moved them (with the 94 OGS API bots: 138) to `sgf/bot_games` and stripped
  their rows from all round data (backups `.bak`). The KGS 2019-03-12 game where *both* sides showed an AI-like span
  had ELFv2v2 as Black: the engine-likeness signal had found a real bot.
- **Only fully analysed games anywhere** (the user's rule): partial games are left out of totals, tables, charts and
  the viewer. All current games are fully analysed (every move has the engine and every human profile).
- Leela alongside KataGo and KataGo-Human added to the plan (R3d).

## Difficulty needs searched losses (2026-11-23, the user spotted move 23 of KGS2003_2003-01-19-3)
Position Difficulty (9d) said 59 % there: the 9d profile picks P8 59 % of the time, and the 8-visit search gave P8
one visit, whose "loss" of 2.92 points is a single raw network value — so a 9d's natural move counted as an error.
Fix: a candidate's loss counts only with >= 4 visits; P(error) needs reliable losses for >= 60 % of the 9d profile's
probability, else the position's difficulty is "–" (not enough data). On the 51,373 moves so far only 42 % then have a
difficulty: the candidate searches (`features.py --candidates 3`, each 9d-likely move searched on its own) are
required, now with 16 visits each (`rounds.py --cand-visits 16`), from the next round.

### Full-move round 1 (2026-11-23 ~22:55)
273 games, every move analysed (17 profiles, 8 visits), 8,708 s. Rank model MAE per single game: overall 2.66
(500 player-games), opening 2.93, middle 2.81, endgame 3.10 (353). The boundary script then killed itself (its own
command line matched its kill pattern — again; use exact PIDs only) before testing candidates; done by hand: 2 games,
16 visits per candidate search (131 / 246 searches), ownership on every turn, 0 failures, 90 s. Rounds restarted from
round 2 with `--candidates 3 --cand-visits 16`.
- **Chunked rounds (the user's request):** each round is extracted in chunks of 40 games; after every chunk the four
  phase models retrain, the learning curve gets a row (column `chunk`), and engine-likeness, difficulty, the report,
  the charts and the viewer refresh — results every ~20–25 min instead of once per round.
- Incident: the boundary script had not died after all; with the hand-started driver, two drivers ran round 2 at
  once, writing the same files. Both stopped (exact PIDs), the round-2 files deleted (unreliable), one chunked driver
  restarted. 5,390 games remain eligible after the bot purge.

## Coordinator + workers (2026-11-23, the user's design: one process per machine doing KataGo, Human and Leela)
`gorank/coordinator.py` (HTTP, stdlib, shared token; tasks of 8 games; silent workers' tasks re-queued after 45 min;
a task failing 3 times is dropped) runs inside `rounds.py --serve`, which also starts a local worker for this PC.
`gorank/worker.py` runs on every machine: fetch a task, analyse its games locally (KataGo search, candidate searches,
ownership and KataGo-Human from one KataGo process; Leela to come, R3d), upload only if every game is fully analysed.
Tested (tests/test_coordinator.py: auth, hand-out, empty queue, failure re-queue, timeout re-queue). Laptop setup and
the WSL port forwarding are in the README. This PC switches to `--serve` after round 2 chunk 1.

## Players 15k–9d only (2026-11-23, the user's rule)
Cells are now 15k…9d (24 ranks x 3 eras); a game with any player weaker than 15k is left out entirely (rounds,
report, charts, viewer); the 20k / 17k human profiles remain as model inputs. 4,172 stage-1 games stay eligible.
Round 2 chunk 1 (first incremental update, 40 KGS games with 16-visit candidate searches): MAE overall 2.52 (580
player-games; round 1: 2.66), opening 2.73, middle 2.65, endgame 3.04. The driver now runs in coordinator mode
(`--serve`, port 8765) with this PC's worker; laptops can join any time.
- **Every move gets a Position Difficulty (the user's requirement):** Game Difficulty is the average over *every* move,
  so candidate searches now continue until >= 95 % of the 9d profile's probability has a reliable (>= 4-visit) loss,
  up to 8 searches of 16 visits per position (before: at most 3). Applies to every new task; older games get the
  top-up pass. (The 9d profile itself was always computed for every move; the gap was KataGo losses for its moves.)

## Difficulty: candidate-selection bug, stricter coverage (2026-11-23, the user noticed many 0 % / "–")
Measured: round 1 (no candidate searches) 59 % of moves "–", 34 % at 0 %; round 2 (16-visit candidates) still 41 % "–",
41 % at 0 %. Cause: a move the 8-visit search had touched once counted as "evaluated", so it got no candidate search —
then its 1-visit loss was rejected as unreliable. Fixed in features.py (only >= 4-visit moves count as evaluated;
every 9d move with >= 2 % gets its own 16-visit search). 0 % is often genuine (every move a 9d would play loses < 2
points), but with 60 % coverage the unsearched — often worse — moves dropped out and biased toward 0 %: a difficulty
now needs reliable losses for >= 90 % of the 9d profile's probability, else "–". Older games get a candidate top-up
pass later (R3b-topup).

## Gains (negative losses) are information (2026-11-23, raised by the user)
Of 61,240 moves with a loss, 28 % are negative (mostly small: two independent 8-visit estimates differ by noise) and
5.0 % (3,032) are gains of >= 1 point: the player's move left a better position than KataGo's shallow search expected —
a move it did not consider or could not read, or noise. Plan R3g: keep the raw value (viewer: "gain"; Avg Point Loss
counts 0), re-check every gain >= 1 point at 200–400 visits before and after the move (~2 h on the A770 for the current
3,032), and keep confirmed finds as a feature for the rank model and for Engine Likeness.
- Decision (the user): no separate deep re-check of gains — at higher playouts they vanish; the R3c visit ladder
  handles them. Gains stay raw (viewer "gain", Avg Point Loss counts 0).

## Chunk-tag bug: 40 games overwritten (2026-11-23)
On the switch to coordinator mode the driver resumed round 2 with its chunk counter at 1, and its first chunk (the 11
KGS games left after the 15k filter) was written as `r002_c01_kgs`, overwriting the finished 40-game chunk (MAE n
580 -> 522). Those 40 games return to the pool (reconcile counts only what is on disk). Fixed: the chunk counter starts
after the highest chunk already on disk for that round.
Speed with full 9d coverage (up to 8 candidate searches of 16 visits per position): 8 games in 831 s = 104 s/game
(before: 42 s/game with <= 3 candidates). Watching the measured coverage before deciding whether to tune.

## 9d coverage with full candidate searches (round 2 chunk 2, 40 OGS games, 9,255 moves; 2026-11-24)
Positions with a Position Difficulty (9d): 79.5 % at the 90 % coverage bar (round 1: 41 %). Of the 20.5 % left, 68 %
have 80–89 % coverage and hit the 8-search cap: flat positions where the remaining 9d probability is spread over many
moves of < 2 % each. Bar lowered to 80 % (-> ~93 % of positions rated); the uncovered tail is moves a 9d rarely plays.
Searching further (12+ per position) would cost another ~40 % time for that tail. 0 % stays common (35 %): genuine —
in most positions no move a 9d would plausibly play loses 2+ points. MAE after the chunk: overall 2.49 (n 596),
opening 2.77, middle 2.58, endgame 3.05. Speed 90–105 s/game.

## R3e, first measurement: how far shallow losses can be trusted (round 2 raw, 2026-11-24)
Same move, same position, the loss from the 8-visit search's shallow look vs. its own 16-visit candidate search:

| Visits behind the shallow loss | Pairs | Mean shallow − deep | Median abs. diff | 90 % abs. diff | Error call (>= 2 pts) flips |
|---|---|---|---|---|---|
| 1 | 8,012 | +0.21 | 0.52 | 2.29 | 15 % |
| 2–3 | 7,229 | 0.00 | 0.32 | 1.38 | 7 % |

A 1-visit loss is biased (overstates the loss by 0.2 points) and calls the wrong side of the 2-point error line 15 %
of the time; 2–3 visits are unbiased and flip 7 %. This supports the display rule (>= 4 visits) and gives the model its
reliability prior: these pairs are kept as training data (R3e). Next: per-tier loss features with visit counts as
inputs in the training set, and the same comparison against the R3c ladder levels once they exist.

## Round 1 re-analysed with the full candidate searches (2026-11-24)
Round 1 (273 games, before the candidate fix) leaves 45 % of all moves without a Position Difficulty (9d); round-2
games reach ~93 %. Rather than a separate top-up mode (R3b-topup: would save ~3 h but needs a second code path and
mixes methods), round 1's files moved to `rounds_full_archive_r001/` (kept: its 1–8-visit data stays available for R3e),
and its cells refill through the normal rounds with the current method — every counted game analysed the same way.
Cost ~7.5 h of GPU at ~100 s/game. Until then the report, charts and viewer show round-2 games only.

## Worker: one unusable game no longer sinks its task (2026-11-24)
Task t000014 came back with 7 of 8 games: features.py skips a game it cannot use (not 19x19 or too short) and the worker
treated the task as failed — the coordinator would have retried it three times and then dropped all eight. The rule is
per game ("fully analysed or nothing"), so the worker now uploads the fully analysed games (rows filtered to them) and
names the skipped ones; those return to the pool. The local worker is replaced after its current task; the task it had
just taken is re-queued by the 45-minute timeout.
- The retried task t000014 came back "ok (1 skipped)". Cost of the restart: the task the old worker had just taken
  waits for the 45-minute timeout (GPU idle meanwhile). Fixed for next time: a worker asking for a task while it still
  holds one has abandoned it (one task at a time), so the coordinator re-queues it at once (test added; 22 tests pass).
  Takes effect at the next driver start.

## Round 2 chunk 4 (2026-11-24)
Training set now only the round-2 games (round 1 archived for re-analysis): 242 player-games. MAE overall 2.09,
opening 2.43, middle 2.21, endgame 2.61 — not comparable with the 2.48 before (different, smaller set; CV noise is large
at this size). Position Difficulty (9d) now exists for 90.5 % of all moves in the report (27,360 moves).

## Position-difficulty rank estimate, first try (gorank/position_elo.py, 2026-11-24)
Idea (Position Elo + Bradley–Terry, own implementation): every position is an opponent whose strength curve is
P_r(error) = P(rank-r human profile loses >= 2 points there); the player's rank is the r maximising the likelihood of
their own error / no-error outcomes. No training. Round 2 (295 player-games, >= 10 rated moves):
raw estimate MAE 4.86, bias −2.44 (players look weaker than declared: their own loss is root-to-root at 8 visits,
noisier than the searched candidate losses behind the curves, so "errors" are over-counted); correlation with the
declared rank 0.70; linearly calibrated (5-fold CV by game) MAE 3.41 vs 5.19 for predicting the mean. Weaker than the
ridge model (~2.1–2.5) on its own, but an independent signal: next, add it as a feature (model G) and measure.
- **Model G = F + the position-difficulty estimate** (`train_rank.py --extra position_elo.csv`, reported only, not
  exported). Round 2, 276 player-games, CV by player: overall best F 2.10 -> best G 2.03 (RMSE 2.62 -> 2.56, within ±1
  28 -> 30 %); middle game F 2.14 -> G 2.15 (no gain). A small gain at the edge of the noise for this sample size;
  re-measured as rounds add data before it becomes the exported model (it would need position_elo in estimate.py too).
- Round 3 chunk 2 (2026-11-24): 436 player-games; MAE overall 1.98 (first time under 2), opening 2.30, middle 2.04, endgame 2.63.
- Round 3 chunk 3: 502 player-games, MAE overall 1.90, opening 2.21, middle 1.97, endgame 2.47.
- **Model G re-measured on 502 player-games:** best F vs best G — overall 1.86 vs 1.84, opening 2.21 vs 2.21, middle
  1.97 vs 1.96, endgame 2.47 vs 2.45. The position-difficulty estimate adds almost nothing on top of the human-policy
  likelihoods (both read the same human net). Decision: not adopted; kept as a reported experiment. Its standalone
  value (no training needed, MAE 3.4 calibrated) may still matter for ranks/eras with little training data.

## Games and accounts flagged by the user (2026-11-24)
The user reviewed games in the viewer and flagged suspected AI use: [flagged game] (OGS, 2022-01-21: very low point loss,
dodges the full-AI pattern), [flagged game], and the account [flagged account] (40 games in stage 1,
KGS 2017-11 to 2019-04, plus one OGS game under the same name; 495 games in the KGS index). New: `gorank/exclusions.py`
reading `~/gorank-data/excluded.tsv` (game or player entries, with reason and date); rounds.py never selects them;
purge_games.py moves them to `sgf/excluded/` and keeps their analysis as labelled positives for the AI-usage model
(`~/gorank-data/labels/suspected_ai_*`, R6). Applied at the next chunk boundary with a driver restart (which also brings
in the coordinator's restarted-worker fix).
Lesson for R6: a cheat that avoids long top-move runs (low loss, few exact engine matches) is not caught by the
current Engine Likeness, which rewards matching KataGo's first choice; low loss relative to the declared rank and to
the position difficulty is the stronger signal.
- 2026-11-24: the user also flagged the account [flagged account] (5d; opponent in [flagged game]). Added
  to excluded.tsv; applied with the others at the next chunk boundary.
- Report fix: report.py had failed since ~04:00 (tables, charts and viewer totals stale) on a game with a "pro" rank
  it could not sort. Amateur-server games with a pro player are now left out like sub-15k ones (outside 15k–9d);
  professionals get their own tier from a dedicated source (PLAN R2b, the user's request).
- Viewer: Era dropdown (All / Pre-AI ≤2015 / Early AI 2016–18 / Current AI 2019+) next to the rank band dropdowns,
  on the Overview, Ranks and Games tabs; report.csv carries per-era band and rank rows (column `era`; charts use
  the all-era rows, unchanged).

## Pro games: sources (2026-11-24, the user's suggestion: kifubara.app, kifudepot.net)
kifudepot.net: robots.txt only blocks similar-game search pages; each game page embeds the full SGF (PB/PW, BR/WR as
dan grades, KM, DT, GN = event); listings by date back to 1949 and by player. No terms forbidding use found (about.php
is a disclaimer). Crawl politely: <= 1 request per 5 s, cache every page. kifubara.app: `Disallow: /games/*/sgf` for
all agents — respected, not used for SGFs. Tiers from goratings.org (robots: Allow /).
- **Effect of removing the user-flagged games** (8 of them were in the training data, 16 player-games; offline
  retrain, same CV): overall MAE 2.00 -> 2.00, opening 2.36 -> 2.43, middle 2.14 -> 2.17, endgame 2.61 -> 2.66. No
  measurable change (16 of 640 player-games; differences within CV noise). Removal still stands: the point is not
  to teach the model that assisted play is normal for a rank, which matters more as data grows and for R6.

## Second worker: the CachyOS laptop (GTX 1650 Ti 4 GB, 2026-11-24)
Set up over SSH (key auth; login shell is fish, so remote scripts run under bash). KataGo 1.18.1 OpenCL (1.18.2 has no
Linux OpenCL build — the setup script now takes the newest release that has one), same b18 and human networks.
Benchmark (default_gtp.cfg; the analysis cfg lacks numSearchThreads): 113–131 nnEvals/s vs ~253 on the A770, so
~+50 % throughput. No Windows port forward needed: a reverse SSH tunnel from WSL (`-R 8765:127.0.0.1:8765`, reconnects
in a loop) gives the laptop's worker `http://127.0.0.1:8765`. Worker started with setsid/nohup (no tmux), log
`~/gorank-worker.log` on the laptop.
The user-flag purge ran: 42 games to sgf/excluded, their rows kept as labels; driver restarted in coordinator mode.
- **Third worker: laptop 2** (Ubuntu, GTX 1060 Max-Q 6 GB; everything in ~/GoRanks, the user's own ~/katago left
  alone): 65–81 nnEvals/s, about a third of the A770. Three machines ≈ 1.8x the A770 alone. Own reverse tunnel.
- **Pipelined chunks:** with three machines of different speeds, the fast ones idled at every chunk end waiting for
  the slowest task. The next chunk is now queued as soon as the current one is fully handed out (training and the
  report refresh run while the workers continue). Tasks 8 -> 4 games and the coordinator timeout 45 -> 90 min: an
  8-game task on the 1060 would take ~40 min, too close to the old timeout (a duplicate run). Driver restarts at the
  next chunk boundary (when the chunk's file is written, before the old code queues the next chunk: nothing in
  flight is lost).
- **GPU care on the laptops (the user's request):** worker.py `--rest 90 --rest-every 2 --max-temp 85` — a 90 s rest
  after every 2 games (mid-task) and after every task, and no new work while an NVIDIA GPU is above 75 C (checked
  with nvidia-smi; the 1650 Ti reports decimals, e.g. 76.88). Tested with a stubbed features.py (groups, rests,
  merge, a skipped game). Both laptop workers switch to it right after their current task (no work lost).
- **Sleep check:** processes survive SSH logout on both (KillUserProcesses=no). Laptop 2 (Ubuntu/GNOME): sleep and
  suspend targets are masked — it cannot suspend. Laptop 1 (CachyOS/KDE): no auto-suspend on AC, but the lid action
  is KDE's default (sleep); a sleep inhibitor cannot be taken from SSH (polkit), so the user sets "lid closed: do
  nothing" on AC in KDE's Power Management.

## KGS and OGS ranks are different scales (2026-11-24, raised by the user)
Human-net strength estimate (model A, the same ruler for both servers) minus the declared rank, overall phase:
KGS covers only D1-3 / D4-6 / D7-9 (the u-go.net archive holds games with a 4d+ player), OGS covers K1..D6 — the
servers overlap only at 1–6d, so a pooled table confounds rank with server. Where they overlap the same declared rank
differs by ~1 rank (1–3d pre-AI: KGS −0.2, OGS +0.5; 4–6d early AI: KGS −0.6, OGS −1.5; n 10–28 each, noisy), and KGS
drifts by era (7–9d: pre −3.8, early −2.4, current −1.7). The human-net ruler itself compresses the scale (kyus read
stronger, high dans weaker), so only differences between servers at the same declared rank are meaningful here.
Changes: report.csv has a `server` column and band/rank rows per (era x server) with "all" for each; the viewer has
a Server dropdown (All / KGS / OGS, Pro when present) beside Era and Rank band, plus a legend note; charts stay all/all.
The rank model already learns a server x period offset (model E/F); next: report its error per server, and
(PLAN R1) a common scale needs overlapping data — KGS kyu games are not available in bulk (the archive is 4d+, per-user
crawling is off-limits), so the human net is the bridge for now; Fox (both kyu and dan, one server) is the candidate
third source.
- **GPU temperatures measured** (nvidia-smi every 10 s, ~40 min, 239 samples each): 1650 Ti mostly 70–80 C under
  load, peak 80.8 C (slowdown 97 C, shutdown 102 C, design target 75 C; 0 s thermal throttling, power-capped
  instead); 1060 Max-Q never above 54 C (slowdown 99 C) — power-limited to ~22–24 W, flags "SW thermal slowdown" at
  48 C (a laptop platform power/thermal policy, not GPU heat). Policy from the data: the temperature guard pauses only
  above 85 C and resumes below 75 C (hysteresis; before, any reading above 75 C paused, a heat/cool cycle at every
  check), rests shortened to 60 s (every 2 games and after each task), keeping the user's rest-and-cool structure.
- train_rank.py: players outside 15k–9d are skipped in training (one amateur-server 'pro' player-game had slipped into the round-3 training set as band Pro, n 1).
- kifudepot listing stopped early at page 1837 (82,620 games, back to 2009 only): a page that failed 4 times was taken as the end of the list. Now a failed page backs off 5–20 min and retries; resumed (cached pages reused).
- Second stop at page 1837: the site had returned the page without its game table (8.8 KB, no rows) and it was cached. Now a page without games is not cached, retried up to 3 times (1–3 min apart), and the crawl stops only after 5 such pages in a row.

## Pro tier: names matched, types per era (2026-11-24, on the first 90,000 listed kifudepot games)
goratings: 2,092 pros, 351,940 rated games (crawl ended at id 2921 after 300 missing ids). Name matching by shared
game dates works for established pros (Park Junghwan 1,406 of 1,428 dates, Iyama Yuta 1,012 / 1,070, Lee Sedol
1,004 / 1,009); 205 goratings players had been claimed by several kifudepot names (rarely recorded pros sharing
tournament dates) — now one name per player, the best-supported one: 1,339 of 2,733 names matched. Dry-run
selection (130 per cell = 100 + 30 % spare, at most 5 games per player per cell) — candidates / players picked:
pre: top 22,657 / 58, mid 40,787 / 112, new 2,905 / 106; early: top 7,284 / 43, mid 20,302 / 100, new 868 / 62;
current: top 17,105 / 46, mid 51,150 / 109, new 1,892 / 76. Every cell has enough. Final selection after the listing
finishes (adds games before 2009).
- **Pro games in the pipeline:** fetched SGFs get BR/WR rewritten to 1p / 2p / 3p = new / mid / top pro at that
  date (goratings, both players where matched; "?" = not matched, skipped like an unranked player), since the
  kifudepot dan grades (九段) say little about strength. report.py reads 1p/2p/3p as 10/11/12 ("Pro new/mid/top",
  band Pro) and keeps games from the pro source (amateur-server "pro" ranks stay out); viewer_export likewise; the
  amateur rank model (train_rank) still trains on 15k–9d only — pros need their own scale (goratings Elo), next step.
  rounds.py: the catalogue reads sgf/pro/pro_players.tsv (cells era x pro_type); every chunk gets up to 15 pro games
  (PRO_PER_CHUNK), interleaved over the 9 cells so they fill evenly (tested).
- Viewer: a 'How the data is made' legend key (open by default) under the Overview, Ranks and game tables — sources and cleaning, how each move is analysed, every column (the user's names) and per-move term, reading the tables; replaces the short legend line. All stage-1 games are analysed: the driver has no round limit (--rounds 1000) and runs until the pool is empty (amateur pool + round 1 re-analysis + pros).

## Visit ladder on rented GPUs (vast.ai, 4x RTX 5070 Ti, $0.692/hr; 2026-11-24)
KataGo 1.17.2 TensorRT build for CUDA 13.2 (unpacked AppImage — no FUSE in containers) with TensorRT 10.16 libraries
(pip wheel fetched straight from pypi.nvidia.com; PyPI and its China mirrors were too slow or lacked it); networks
uploaded from this PC. Benchmark (b18, one GPU): ~4,300 nnEvals/s (~17x the A770).
Design, after three failures: (1) the Python driver could not parse the deep searches' progress JSON fast enough —
KataGo blocked on its output and the GPU idled; (2) one separate search per level costs the sum of the levels; (3)
mid-search reports with full variations were ~255 MB per game. Now KataGo reads a prepared query file
(ladder_queries.py) and writes through a C filter (ladder_filter.c, string scanning, 255 MB -> 40 MB in 0.3 s) that
keeps only the first report past each level plus the final one, full variations kept; no Python while the GPUs work.
Levels (the user's call, after timing one game at 5k: 175 s per game per GPU, ~65 h for the pool): max 1k —
10 / 100 / 250 as their own tiny searches, 500 / 1000 as snapshots of one 1k search per move (measured snapshot visits
e.g. 530 and 1,012). Per batch of 20 games one KataGo per GPU (supervisor programs gorank-ladder-0..3), output gzip
-1; this PC pulls finished batches every 5 min (pull_ladder.sh), verifies gzip, then deletes them on the instance.
4,121 amateur games in 207 batches; pros follow once fetched. Next: the parser (orjson, 128 cores) and the candidate
second pass, then retraining on the ladder data.
- 2026-11-24: the home 8-visit KataGo queue (driver, A770 worker, both laptop workers) stopped by exact PIDs / names: the vast run covers KataGo + KataGo-Human at 1,000 visits each (the user's rule: every engine exactly 1k visits). Home GPUs move to Leela at 5,000 playouts (the user's split). Finished chunk data kept.
- **Final split (the user's calls, 2026-11-24):** vast — KataGo engine 1,000 visits (+ ownership) and the 17
  KataGo-Human profiles at 100 visits each (human-imitation search settings from gtp_human5k_example.cfg, plus the raw
  human policy), full variations; home GPUs — Leela 0.11 at 1,000 playouts (lz-analyze got a `playouts N` option in
  Gifu's Leela: a fixed-size search that stops by itself and prints its final lines), full variations, through the
  coordinator (leela_feed.py + worker.py Leela mode), laptops keep their rests. 207 vast batches, 1,031 Leela tasks,
  4,121 amateur games each.
- Incident: an SSH command on the instance (a `pkill -f` pattern) dropped the connection and SSH refused for ~2 min;
  the instance itself stayed up. Rule from now on: on remote machines processes are stopped only through supervisor
  or by exact PID / exact process name (`pgrep -x`), never by `pkill -f` patterns.
- Engine Likeness is now shown as a percentage (the user's request): engine-like moves as a % of the player's eligible moves (move 41 on, not decided), column 'Engine Likeness %' in report, charts, viewer (tables, band charts, Games list) and legend; CLAUDE.md column list updated.
- **Continuous retraining restored (the user caught that it had stopped):** it stopped when the home 8-visit queue
  was shut down. Now `ladder_parse.py` turns each vast batch (KataGo 1,000 visits + 17 human profiles) into the same
  training rows and raw records via features.Job (first batch: 20/20 games, 4,010 rows, 49 s; pass moves count as
  covered), and `vast/retrain_loop.sh` retrains the four phase models after every new batch (models in
  ~/gorank-data/models/vast, learning curve in rounds_vast/learning_curve.log) and refreshes report, charts and viewer.
  train_rank `auto=` specs take each game's server from its file name (batches mix KGS and OGS).
- Tough Spots is now the average per player-game and Key Move Found % the average of each game's found share (games with any Tough Spot) — the user's rule; totals kept as hard_total (used for the >= 5 threshold of the found-% charts).

## Public site and data (2026-11-24, the user's request)
GitHub Pages from the Gifu repository's `gifu-training` branch (created from main in a separate worktree,
/mnt/c/Gifu-training, so the uncommitted work on main is untouched): https://jjldonley.github.io/Gifu/ — the viewer
under docs/; GoRanks code, plan, reports, charts and per-batch training rows under GoRanks/; Gifu's Leela with the
lz-analyze `playouts` option under leela/src. Raw KataGo output per batch (~360 MB each, too big for git) in the
release `gifu-training-data`. `gorank/publish.py` builds and pushes it; `gorank/vast/publish_loop.sh` republishes
every 250 newly analysed games. Privacy: player names -> salted anonymous ids (P-xxxxxx), the user's flagged games /
accounts redacted from the reports and never published, per-game chart images (names) not published, no pro records.
- Bug (mine): the Accuracy / Match code reused the variable k, which is the player's profile index in ai_scores.score_game; the per-move scoring crashed (IndexError), so since that change no report/viewer refresh had completed (site and viewer showed the old counts). Fixed (rank_played).
- Labels (the user's request): 'Tough Spots (per game)' and 'Key Move Found % (per game)' in the viewer, report, Discord images and legends (Key Move Found % (per game) = each game's found share of its Tough Spots, averaged over the games); published as e4780e7 with the 1,000-visit data (311 games).
- **Pro selection, final** (full kifudepot list: 127,045 games back to 1949; 1,467 of 3,179 names matched to
  goratings): 130 picks per era x type (100 + 30 % spare, <= 5 per player per cell), 46–115 players per cell; pre-AI
  now has far more candidates (top 55,354, mid 59,823, new 5,081). The listing crawl had kept requesting past the last
  page (the site repeats it) up to page 5,000 — ~2,000 needless requests at 1 per 5 s; it now stops after 5 pages
  without new games. Fetching the ~1,170 picked SGFs (1 request per 5 s, ~1.6 h); then they join the vast and Leela
  queues as their own batches.
- **Leela parsed** (`leela_parse.py`: Leela's best, the played move's rank, visit share — Accuracy / Match as in Gifu —
  and win-rate loss per move): 64,924 moves, 324 games. On the 9,700 moves both engines have so far, players chose
  Leela 0.11's first choice 53.4 % of the time vs KataGo's (1,000 visits) 34.8 % (both 28.4 %): Leela, trained on
  human pro games, is far closer to human play — the reason to use it for human-likeness and early-AI-era cheating.
  The rank-model test (F + Leela features, CV by player) waits until the vast pass completes (~10 h), when every
  Leela game also has its KataGo rows (the two passes run through the games in different orders).
- Decision (the user): Leela is not a second opinion on correctness — KataGo is far stronger. Losses / mistakes / difficulty stay KataGo + human profiles; Leela is used for early-AI-era cheat detection, as a human-style strength feature (if it helps in CV), and for era comparison. PLAN R3d updated.
- Publishing (the user's call): real player names and the analysed amateur SGFs (4,121, bot-checked, 15k–9d, not flagged) are published — names help identify cheaters and bad games. Still never published: the flagged games / accounts and suspected-AI labels, pro records.

## Positional play, symmetry, forced errors (2026-11-24, the user's ideas)
- **Areas at the 4th line** (4 corners, 4 sides, center): per move the area played and KataGo's best area; Same Area %,
  Direction Errors (2+ pts lost, other area), Local Errors (2+ pts, same area), Resolved-Area Play (into an area with
  >= 85 % mean ownership certainty around the point while the best was open), Lower-Value Area (other area, 1+ pt lost).
  Batch b0000 (3,507 moves): same area 54 %, direction 19.6 %, local 10.4 %, resolved 6.3 %, lower-value 25.9 %.
- **Symmetric positions:** a move equivalent to KataGo's best under a board symmetry that leaves the position unchanged
  counts as the best (D4 vs Q16 on an empty board) — in every agreement figure.
- **Forced errors:** forcing = KataGo's best reply within 2 points of the opponent's move and the best move elsewhere
  losing 3+ (or none considered); Forced Error % = answered locally but lost 2+; Forcing Success % = own forcing moves
  after which the opponent lost 2+. Two batches: 13 % of moves face a forcing move; error rate on them kyu 23.1 %,
  dan 14.4 % — a rank signal to test in the model.
- Viewer: Positional play table (Overview), Where they play (game view: 4th-line distribution map, liked = 10 %+ of
  the best's visits else greyed by loss; per-area table), Ownership overlay on the board, legend entries.
- KataGo symmetry study (lightvector, July 2026, "How Symmetric Are the Insides of a Go Network?"): networks learn
  approximate symmetry; supports (a) averaging over the 8 symmetries for the human-profile policies and (b)
  symmetry-disagreement as a hardness weight for Engine Likeness — both planned, to be measured first.
- **Shapes, layer 1** (`gorank/shapes.py`, the user's idea): 21 named shapes from the stones around each move
  (capture, atari, connect, cut, wedge, crosscut, empty triangle, hane, tiger's mouth, bamboo joint, extend, attach,
  peep, kosumi, one-point / two-point jump, knight's / large knight's move, 3-3 invasion, approach, open), rotation /
  mirror / colour invariant; 6 tests. 399 games: the played move has the same primary shape as KataGo's best in
  34.1 % (kyu) vs 44.7 % (dan) of moves. KataGo's best is far more often a 3-3 invasion (1.1 % vs 0.1 %), an
  attachment (6.9 % vs 3.9 / 4.8), a peep (3.5 vs 1.7 / 2.1), a large knight (2.1 vs 1.0 / 1.1); kyus connect (14.2 %),
  crosscut (6.7 %) and make empty triangles (2.3 %) more than dans or KataGo (1.8 %). Per move: shape, shape_best,
  same_shape, bad_shape; report: Same Shape %, Empty Triangle %; viewer: Shape / Best Shape under the board.
  Chess background: Stockfish's classical eval (named concepts), lichess puzzle themes (rule-based tagging), concept
  probing in AlphaZero (McGrath et al., PNAS 2022; Schut et al. 2023), look-ahead in Leela Chess Zero (Jenner et al.
  2024); Go: Stern et al. 2006, Coulom 2007 pattern ratings.
- Background loops restructured: retrain_loop.sh and publish_loop.sh waited on each other (publish waited for a moment with no refresh running, which never came, so the site stalled at 21:23; and retraining had slowed to one per ~20 min behind full refreshes). Now one loop, gorank/vast/pipeline_loop.sh: parse + retrain after every batch (~1 min), one report/viewer refresh every 30 min, then push the site when 250+ more games are in; raw-batch uploads to the release run separately (upload_raw.sh).
- The user asked to stop releasing the training data: the GitHub release gifu-training-data (90 raw KataGo files) and its tag were deleted, the upload step removed from pipeline_loop.sh (upload_raw.sh deleted). Raw output stays on this PC only.
- Report refresh: ai_scores now runs one process per raw file on 12 of the 16 cores and the results are concatenated (rounds.score_parallel); single-process re-scoring had grown to ~30 min at ~1,400 games with shapes, delaying the site.
- Games Analyzed now counts each game once (both players together; the user's rule) in the report ('games' field; 'player_games' kept for the per-player averages), viewer, Discord images and the site header. Model sample sizes are player ratings (two per game), now reported as 'ratings from N games'.
- **Site moved into the Gifu folder (the user's request):** the site is built in Gifu/GoRanks/site and pushed onto the
  `gifu-training` branch straight from the Gifu working folder with a temporary git index (`gorank/push_site.py`; the
  main checkout, its index and uncommitted work untouched; no separate /mnt/c/Gifu-training checkout any more). Pages
  now deploys GoRanks/site through a GitHub Actions workflow (GoRanks/ci/goranks-pages.yml -> .github/workflows on the
  branch; Pages build_type switched to workflow). First deploy failed on a GitHub timing glitch (artifact not yet
  visible 0.3 s after upload), the re-run succeeded: live data generated 2026-11-25 00:00, 1,488 games.
- Summary data slimmed from 19 MB to 3.9 MB (no per-area fields in band / rank rows, only each game's Overall rows;
  a game's phase rows ship with its own file). "Where they play" redesigned (two maps side by side — the player's
  moves and KataGo's best — area shading, a board-shaped 3x3 area table, labelled stats, a colour key per player;
  the White key showed a black stone, fixed). Open: game files ~212 KB each (ownership per move) -> ~1 GB at 5,400
  games, over the Pages limit: compress ownership (run-length) or move the site to one SQLite file.

## Do the new inputs lower the rank error? (2026-11-25, 6,176 ratings from 3,088 games, 1,000-visit data, CV by player)
Model F alone: MAE 1.995, RMSE 2.710, within 1 rank 33.8 %. F + all new per-player-game inputs (same area, direction /
local errors, resolved-area play, same shape, empty triangles, forced errors, Accuracy, Match): MAE 1.995, RMSE 2.704,
33.8 -> 34.1 % within 1. Each group alone: 1.994–1.995. No measurable gain: these measures do differ by rank (forced
errors kyu 23 % vs dan 14 %, same shape 34 vs 45 %), but the 17 human-profile likelihoods already carry that
information — they predict the very moves these measures summarise. The ~2.0 plateau is set by single-game noise
(one player's ~100–150 moves per game) and label noise (declared ranks themselves are uncertain by about a rank,
KGS vs OGS scales). The inputs stay as analysis on the site; for the rank estimate the next lever is several games
per player (the account view, R10), not more per-move features.

## The per-game estimate is a performance measure (2026-11-25, the user's point; 6,168 ratings from 3,088 games)
In games between players within 1 declared rank, the side the model rates higher relative to its declared rank WON
68.9 % of 1,192 games (50 % = no signal) — the estimate tracks real performance. Estimate minus declared: winners
+0.09, losers -0.09. Residual variance 7.34 = 3.26 within a player (game-to-game swing, sd 1.8 ranks, 398 players with
3+ games) + 4.08 between players (declared rank off for the player — under/overrated accounts, KGS vs OGS — or a
style the model misreads). So scoring each game against the declared rank counts genuine over / under-performance as
error. Plan: (1) name and show the per-game estimate as a Performance Rank; (2) a player's rank = mean performance
over their games; (3) evaluate by win prediction in even games and by multi-game MAE as well as per-game MAE;
(4) result-adjusted training targets (Elo-style performance from result vs the opponent's rank), kept only if (3)
improves.
