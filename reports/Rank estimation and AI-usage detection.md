# Calibrate rank and engine-likeness on measured baselines, and use the OGS dump instead of crawling KGS

Gifu's rank model and AI-usage estimate (PLAN §11) are well aimed, but the research changes four things. **First, the data plan has to change.** The per-user KGS game archive now shows a login form, and KGS's robots.txt disallows `/gameArchives.jsp` (both observed 2026-11-23). Crawling it "walking from opponents' names" is not acceptable. Meanwhile, **OGS has authorised a public dump of 27 M (2021) and about 56 M (2025) games**, and its raw JSON carries each player's rating, deviation and bot flag *at game time* plus per-move milliseconds ([za3k OGS readme](https://za3k.com/ogs/readme.txt); [za3k.com/ogs](https://za3k.com/ogs)). A 21 M-game Fox corpus with every rank from 18k to 9d also exists ([featurecat/go-dataset](https://github.com/featurecat/go-dataset)). KGS in bulk only covers games with a 4d+ player, and only up to April 2019 ([u-go.net 4d+](https://www.u-go.net/gamerecords-4d/)).

**Second, the KataGo human-SL profiles are KGS-2020 profiles.** In code, `rank_5k` means "a KGS 5k on 2020-03-01 at 20 min + 5×30 s byo-yomi", and `preaz_5k` is the same with the date 2016-09-01 ([sgfmetadata.cpp](https://raw.githubusercontent.com/lightvector/KataGo/master/cpp/neuralnet/sgfmetadata.cpp)). There is also an opponent-aware `rank_{BR}_{WR}` profile ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)). So KGS is the natural internal scale, and the `preaz_`/`rank_` pair is a ready-made pre-AI/post-AI axis.

**Third, the realistic accuracy target is modest.** The best published Go estimator, which already uses human-SL priors as features, gets 87.5 % exact rank class from 15 random games of a rank but only **63 % when all 15 games come from one player** ([Kuboki et al. 2025](https://arxiv.org/html/2505.00279)). R0 and R7 must be scored per player.

**Fourth, no Go body decides AI use by statistics alone, and no labelled cheater dataset exists.** Chess's best-documented system needs z ≈ 5 to judge on statistics alone ([ECU Arbiters' Corner](https://arbiters.europechess.org/arbiters-corner-april-2025-prof-regans-statistical-system-explained/)). A few well-timed engine moves (2–4) can swing a game while leaving overall match rates almost normal ([Keren 2026](https://arxiv.org/html/2601.05386v1)). Gifu's engine-likeness must therefore be calibrated on clean nulls and synthetic positives, weighted toward critical moments, and expressed as multi-game evidence. That is close to what R5 and R6 already plan. The changes are in the details: priors, critical-move weighting, timing as a separate z, and a player-level combination rule.

## What chess anti-cheating actually does

**It screens first and judges later, on events, not single games.** Regan's system has a quick *screening* test that reports a Raw Outlier Index on a scale with mean 50 and sd 5. It is explicitly "not for judgment": below 60 is "completely normal", 65 is about 1 in 750, and above 70 about 1 in 30,000. Only the hours-long *full* test renders statistical judgments, with "z=2.50 [as a] floor for strong statistical impact amid other evidence" and "z=5.00 as countenancing judgment by statistics alone" ([ECU Arbiters' Corner, Apr 2025](https://arbiters.europechess.org/arbiters-corner-april-2025-prof-regans-statistical-system-explained/)). The test unit is the event. His confidence intervals were vetted on simulated 9-game units, events are combined with Stouffer/Fisher, and sample size is counted in moves: about 1,500 analysed moves "is a fine sample". Book moves and turns 1–8 are excluded, while forced moves and time-trouble blunders are *not* removed ad hoc ([Regan, 20 questions](https://cse.buffalo.edu/~regan/chess/fidelity/RichT20Qs.html)).

**Moves in a game are not independent, so z is deflated.** Regan describes "sparse dependence with exponential decay within a game" and adjusts z 10–15 % downward. The adjustment was vetted "by millions of resampling trials—emphasizing 4-game, 9-game, and 16-game sets" ([Regan, Cross Labs talk 2022](https://cse.buffalo.edu/faculty/regan/Talks/CrossLabsOct2022.pdf)). His criterion for a *safe* model is directly usable as a test: on clean data, its z-scores must follow a normal distribution, or at least stay conservatively within it at the z ≥ 2 end (same talk).

**The prior decides what a z means.** At an assumed in-person cheating rate of 1 in 5,000, a z = 4 (face value 1 in 31,574) still leaves a 1-in-6 chance of a false positive. At an online rate of 2 %, the same z = 4 gives about 600:1 odds, and z = 3.75 about 200:1, which is "OK here, but not if factual rate is under 1%". His target confidence is 99–99.5 % ([Cross Labs talk](https://cse.buffalo.edu/faculty/regan/Talks/CrossLabsOct2022.pdf)). The *look-elsewhere* effect matters too: with about 100,000 player-events a year, natural outliers will occur ([ECU](https://arbiters.europechess.org/arbiters-corner-april-2025-prof-regans-statistical-system-explained/)).

**Telemetry is a second, independent z.** Online platforms record clicks, focus changes and move timing. Regan combines independent quality and telemetry z-scores as z = (z1 + z2)/√2, so two values of 3.5 make about 4.95. A rare *material* behavioural mark (a phone found) makes even z ≈ 2.5–3 meaningful, while an arbitrary, neutral mark is selection bias ([Cross Labs talk](https://cse.buffalo.edu/faculty/regan/Talks/CrossLabsOct2022.pdf)). Chess.com uses "over 100 gameplay factors", closes about 85 % of accounts without human review and sends titled players to a panel. It granted 0.2 % of about 28,000 appeals in Q1 2025, and it runs separate automatic sandbagging detection ([Chess.com Fair Play](https://www.chess.com/article/view/chess-com-fair-play-and-cheat-detection)). Lichess's Irwin analyses engine PVs to "assist moderators", and Kaladin uses CNNs on player "insights" ([irwin](https://github.com/clarkerubber/irwin); [kaladin](https://github.com/lichess-org/kaladin)). Neither documents its features, so claims that they use move times are forum hearsay.

**Selective cheating is cheap and hard to see in averages.** In engine-vs-engine simulation, random single engine moves barely help (score 0.512 with one, 0.585 with five). Moves timed by a value-gap threshold give 0.656 / 0.762 / 0.838 for 1 / 2 / 3 interventions, and an optimised policy reaches about 0.91 with 4. The author notes that detection "rel[ies] heavily on the presence of many excellent, difficult to find moves" ([Keren, arXiv 2601.05386](https://arxiv.org/html/2601.05386v1)). Chess.com's confessions page describes a GM who used an engine only from "around move 17-18" on ([Chess.com](https://www.chess.com/article/view/chess-com-fair-play-and-cheat-detection)). **No published HMM, change-point or per-move-posterior model for partial assistance was found in chess or Go.** The one recent neural paper labels whole games and reports 98 % on an easy human-vs-engine split ([CEUR-WS 3885/13](https://ceur-ws.org/Vol-3885/paper13.pdf)). Gifu's sticky two-state HMM (R5) would therefore be new. The standard modelling reference for persistent states is the sticky HDP-HMM ([Fox et al., arXiv 0905.2592](https://arxiv.org/abs/0905.2592)).

## Go practice: statistics trigger a review, people decide

**Research is thin and single-game.** Egri-Nagy & Törmänen judge one game "solely based on the game record". They use the win-rate curve (a cheater's rises steadily), the mean per-move score change ("average effect") and KataGo top-move matches, plus human cues such as a near-constant ≈ 5 s lag and "safe play" after about 98 % ([arXiv 2009.01606](https://arxiv.org/html/2009.01606v3)). In four reviewed games, the suspected AI users averaged about −0.16 to −0.25 points per move and a strong human about −0.65. They conclude that "automated cheat-detection may never be feasible due to the danger of false positives" (same paper). Park et al. 2022 build an "AI-likeness rate" null from face-to-face KB League professional games and test one alleged game, at about 1 in 10⁶ ([JKPS 81:1189](https://ui.adsabs.harvard.edu/abs/2022JKPS...81.1189P/abstract)). That figure comes from a search summary, because the full text could not be read. **No public dataset of known Go cheaters exists.** An OGS proposal to create one with volunteer cheaters went nowhere ([OGS forum](https://forums.online-go.com/t/creating-a-dataset-to-develop-automatic-cheating-classifiers/36001)).

**Baselines are high and depend on the definition.** In the 2018 PGETC League A case, a move counted as a "match" if it was in Leela's top 3 and within 5 % of the best. On that definition the player's offline games matched 70–80 % and the suspect game 98 % over moves 50–150. Another strong player was seen at 89 % ([Life in 19x19](https://lifein19x19.com/viewtopic.php?p=228865)). The final outcome of the case was not found. Top professionals reach about 65 % on Fox's published match figures ([Sina, secondary](https://sports.sina.cn/others/qipai/2025-02-28/detail-inemzpzn9943912.d.html?vt=4)). A manual reviewer starts to worry near 90 % of non-obvious moves on KataGo's top choice, but warns never to convict on one or two games and that sandbagging opponents inflate match rates ([Nordic Go Dojo](https://nordicgodojo.eu/post/57/how-to-catch-cheaters-online)).

**Bodies decide on review plus physical evidence.** Since 2023/24 the EGF requires recorded video for PGETC League A and youth team events. Its odd-sized Fair-Play Committee compares "the level of play with the nominal strength of the player", and the guide gives no numeric thresholds ([EGF Fair-Play Guide, Oct 2023](https://eurogofed.org/tm/uploads2/PGETC//EGF_FairPlay_Guide.pdf)). SEYGO names the patterns it looks for: AI until the win is secure, AI only in difficult moments, a few AI moves, and "alternating between high level play and a player's current level play". It also says that "playing a few ranks higher than your current ranking is a very common occurrence" and is not cheating ([SEYGO policy](https://www.eurogofed.org/seygo/news/2019/SEYGO_News/SEYGO_Tour_FairPlay_Policy.pdf)). Professional cases were settled by admission or video: Kim Eun-ji, suspended one year in 2020 ([Korea Times](https://www.koreatimes.co.kr/sports/20201123/go-player-cheated-using-ai)), and a 2020 court case over a camera-and-earphone setup ([Asiae](https://www.asiae.co.kr/en/article/2020071519260904451)). The Chunlan Cup 2023 accusation was dropped after weeks of investigation ([ChessBase](https://en.chessbase.com/post/cheating-problem-in-go)). OGS publishes no detection method, but its public game API carries an undocumented `bot_detection_results` field (observed). Gifu's "engine-likeness, never cheated" wording matches this practice.

## Rank from moves: what the numbers are and what the profiles mean

| Work | Data | Result |
|---|---|---|
| Moudřík, Baudiš & Neruda 2015 ([arXiv 1512.08969](https://arxiv.org/pdf/1512.08969)) | KGS, 20k–6d, even games, 10–50 games per sample | RMSE 2.66 ranks (mean-only 7.50) |
| Chen, Shih & Wu, ICLR 2025 ([arXiv 2502.17109](https://arxiv.org/abs/2502.17109)) | Fox, 11 classes 3–5k … 9d | > 80 % exact at 15 games, 97.5 % at 100; ±1 class ≈ 80 % from 1 game; plain SL-policy baselines 49 % at 100 |
| Kuboki et al. 2025 ([arXiv 2505.00279](https://arxiv.org/html/2505.00279)) | Fox ≈ 2017, same classes; LightGBM on strength score + **human-SL `preaz_*` geometric-mean priors at 10 levels** + score-loss stats | random sampling 37.5 / 64.8 / 79.6 / 87.5 / 91.7 % at 1 / 5 / 10 / 15 / 20 games; **one player's 15 games: 63.2 %** |

Three lessons follow. Per-player bias dominates once games are not pooled across players. Human-SL priors and a learned strength score both matter, while engine loss adds little (Kuboki ablation). Server artefacts distort the top and the entry ranks: on Fox, pros and AIs sit in 9d, and new accounts can only start at up to 3d ([Chen et al.](https://arxiv.org/abs/2502.17109)). None of these works covers ranks below 5k, so Gifu's K1–K4 bands have no published benchmark.

**The KataGo human-SL net conditions on much more than rank.** Its metadata includes separate Black and White ranks, unranked and unknown flags, rated or unrated, time-control system and times, date, and a source id. The source constants are **OGS 1, KGS 2, Fox 3, Tygem 4, GoGoD 5, Go4Go 6** ([sgfmetadata.h](https://raw.githubusercontent.com/lightvector/KataGo/master/cpp/neuralnet/sgfmetadata.h)). That these were the training sources is inferred from the constants, since no official list was found. The public profiles fix source = KGS ("KGS rating system is pretty reasonable"), byo-yomi 20 min + 5×30 s, and date 2020-03-01 (`rank_`) or 2016-09-01 (`preaz_`) ([sgfmetadata.cpp](https://raw.githubusercontent.com/lightvector/KataGo/master/cpp/neuralnet/sgfmetadata.cpp)). The analysis engine accepts profile names only, not arbitrary metadata. It does accept `rank_{BR}_{WR}`, with a warning beyond 9 ranks of difference ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)). The value and score heads "are NOT human-like" and are biased by "rank inaccuracies, sandbagging, server pairing effects, disconnections, and resignations" ([KataGo #1030](https://github.com/lightvector/KataGo/issues/1030)). One OGS study used MLE over `rank_20k…rank_6d` as a fixed ruler on 11k filtered games. It found OGS kyu ranks much tougher than the profiles before the 2021 OGS rank change, close to them in 2021–2024, and drifting tougher again in 2024–2025 ([djma/ogs-rank-analysis](https://github.com/djma/ogs-rank-analysis)).

## Data: what exists, what may be fetched, what the fields mean

**KGS.** The u-go.net archives hold games with a 7d+ player (or two 6d players), about 2–19k a year from 2001 to April 2019 ([u-go.net](https://www.u-go.net/gamerecords/)). A 4d+ set covers 2007–2019-04, with about 93–218k games a year to 2015 and about 8–21k a month after that ([u-go.net 4d+](https://www.u-go.net/gamerecords-4d/)). There is no bulk kyu data and nothing after April 2019. Per-user archives sit behind a login, and robots.txt says `Disallow: /gameArchives.jsp` (observed, and noted years ago by scraper authors ([WWW::KGS::GameArchives](https://mirrors.edge.kernel.org/CPAN/modules/by-module/WWW/WWW-KGS-GameArchives-0.01.readme))). The rank markers have fixed meanings:

- **"?"**: the estimate is "little more than a wild guess".
- **"-"**: the player has no rank and wants none.
- **What counts**: only ranked 19×19 games with ≤ 6 handicap stones from the last 180 days ([KGS ranks](https://gokgs.com/help/rank.html); [KGS rating math](https://gokgs.com/help/rmath.html)).
- **Account flags**: `r` for robots and `~` for "plays stronger players far more often than weaker ones" ([KGS JSON types](https://gokgs.com/json/dataTypes.html)). Neither flag appears in the SGF.

SGFs carry BR/WR at game time, TM/OT and per-move BL/WL with OB/OW. During byo-yomi only periods are recorded, so move times are quantised ([Life in 19x19](https://lifein19x19.com/viewtopic.php?p=222957)).

**OGS.** The dump was "distributed with permission of OGS", and OGS has said research use is fine. Its readme warns that "The ranks of the players are likely to be wrong in SGF files", so the JSON should be used ([readme](https://za3k.com/ogs/readme.txt)). From the API (one public game inspected), the JSON has:

- `historical_ratings`, with rating, deviation, `ranking` and `ui_class` per player at game time.
- `ranked`, `annulled`, `handicap`, `komi`, `rules`, `outcome`.
- `time_control_parameters.speed` (blitz / live / correspondence).
- `tournament` and `ladder`.
- `gamedata.moves` as `[x, y, ms]`.

`ui_class` is "bot" for bot accounts (verified on a public bot). Live API use has no published rate limit; users report 429s at about 2 requests/s ([OGS forum](https://forums.online-go.com/t/api-retrieval-of-games/17120)). OGS rank scale history:

- **2021-01-28**: OGS switched to rank = ln(rating/525)·23.15, a "notable bump" for weaker players.
- **Offsets after the switch**: projected OGS 1d ≈ 0.7 stone weaker than EGF 1d and ≈ 0.8 stone stronger than AGA 1d ([OGS 2021 adjustments](https://forums.online-go.com/t/2021-rating-and-rank-adjustments/33389)).
- **Provisional status**: a rank is provisional ("?") while deviation is ≥ about 160. Players leave it after 6.4 games on average and need 12 to reach RD < 120 ([OGS 2020 analysis](https://forums.online-go.com/t/2020-rating-and-rank-tweaks-and-analysis/28649)).

**Conversions.** No paired-account study between KGS, OGS, EGF and AGA was found. The forum consensus for KGS against EGF is "about equal, EGF maybe half a stone stronger", with ± several ranks for any individual ([Life in 19x19](https://lifein19x19.com/viewtopic.php?p=164914)). A 2020 survey of 190 players predates the OGS change ([NTUST survey](https://ntust-mitlab.github.io/go-rank-survey-2020/go-survey-results-2020)). The defensible route is to fit each server-and-period offset onto the KGS-profile scale with the human-SL likelihood itself, then check the result against OGS's published EGF and AGA offsets.

## Cleaning: what comparable datasets did

- **Maia**: only games where both players are in the same 100-point bin; no Bullet; the first 10 plies dropped; moves made with under 30 s left dropped; a held-out month for testing ([Maia](https://ar5iv.labs.arxiv.org/html/2006.01855)).
- **Maia-2**: Rapid games only; the same clock and ply rules; deliberately *oversamples* uneven pairings ([Maia-2](https://arxiv.org/html/2409.20553)).
- **Go studies**: even games only and games played *while at* the rank ([Moudřík 2015](https://arxiv.org/pdf/1512.08969)); ≥ 50 plies, finished by passes or resignation rather than timeout or disconnection ([Kuboki 2025](https://arxiv.org/html/2505.00279)); players below 5k dropped as too volatile ([Chen 2025](https://arxiv.org/abs/2502.17109)).
- **KataGo human SL**: *conditions on* rated-ness, time control and rank uncertainty instead of filtering them out ([sgfmetadata.h](https://raw.githubusercontent.com/lightvector/KataGo/master/cpp/neuralnet/sgfmetadata.h)).
- **Sandbagging**: no published Go sandbagger filter was found. Chess.com treats sandbagging as its own automated detector ([Chess.com](https://www.chess.com/article/view/chess-com-fair-play-and-cheat-detection)).

## Changes this suggests for PLAN.md section 11

1. **R2 data sources.** Drop the per-user KGS crawl: it now requires a login and robots.txt disallows it. Use these sources instead:
   - **Pre-AI**: u-go.net KGS 4d+/7d+ for dan bands.
   - **Kyu bands**: the OGS JSON dump for 2013–2015 (and Fox if its dates allow).
   - **Early AI**: KGS 4d+ 2016–2019-04, OGS 2016–2018 and Fox ≈ 2017.
   - **Current**: OGS 2019–2025-05.

   Record the source and its permission or licence per corpus (OGS: research use OK; Fox set: GPL-3.0). Any live OGS API use stays ≤ 1 request/s with backoff on 429/403. Say plainly that pre-AI kyu data is OGS-only, from 2013–2015, and comes from a server whose ranks were deflated then.
2. **R2 filters, made concrete for OGS JSON.**
   - **Rank at game time**: use `historical_ratings` and drop games where either deviation is ≥ 160.
   - **Drop**: `ui_class` = bot, `annulled`, unranked, correspondence (or keep it as its own stratum), outcomes of timeout, disconnection or cancellation, and fewer than 50 moves.
   - **Handicap**: keep 0–1, and tag handicap games as their own stratum.
   - **Time**: record the speed and per-move ms.
   - **KGS**: drop BR/WR with "?" or "-", and keep a bot name list.
   - **Fox**: drop 9d (pros and AIs) and ≤ 3d games from accounts new to that rank.
3. **R1: KGS-2020 is the internal scale,** because the human-SL `rank_` profiles are KGS ranks at 2020-03-01. Fit offsets per (server, period) with the human-SL likelihood: OGS before and after 2021-01-28 and after 2024, Fox, and KGS by year. Check them against OGS's stated EGF and AGA offsets (≈ −0.7 / +0.8 stone at 1d). Store the offsets in `rank_bands.kv` keyed `<server>.<period>`.
4. **R3 features.**
   - Query `rank_{BR}_{WR}` (the player's rank against the opponent's), not only `rank_X`.
   - Compute both `preaz_*` and `rank_*` likelihoods. Their difference is a direct "AI-era opening" feature and the era fit for undated games.
   - Add Kuboki's geometric-mean prior per level as the main human feature.
   - Store the per-move value gap (best minus second-best and best minus played) so critical moments can be weighted.
5. **R0 and R7 evaluation.** Report exact and ±1-band accuracy *with all games from one held-out player* at 1, 5, 15 and 50 games, beside RMSE in ranks. The comparison targets are Kuboki's 63 % exact at 15 games (dan bands, Fox) and Moudřík's 2.66-rank RMSE. Kyu bands have no published benchmark, so say so.
6. **R5 evidence weighting.** Weight each move's likelihood ratio by its criticality (the value gap between the best move and the human-expected move). Two to four critical engine moves can decide a game while barely moving match rates. Report a "critical-moment" sub-score beside the overall one. Keep timing as a *separate* z combined by (z1+z2)/√2, never inside the move LR. Stratify by speed and treat KGS byo-yomi times as quantised.
7. **R6 calibration made explicit.**
   - **Safety**: on held-out clean games, the per-player statistic must be ≈ N(0,1), and ≥ N(0,1) is not acceptable in the tail (Regan's safety test).
   - **Dependence**: deflate for within-game dependence by resampling whole games in 4/9/16-game sets.
   - **Labels**: user-facing levels map to z bands (< 2 normal, 2–3 "unusual", ≥ 3 "strongly engine-like").
   - **Single game**: never above "unusual"; one game is weak evidence by construction.
   - **Look-elsewhere**: state the effect in the help text.
   - **Synthetic positives**: add "2–4 engine moves at the most critical moments" and "engine only after move N" to the R6 list.
8. **R5/R10 players' multi-game combination.** Combine per-game z with Stouffer, not by averaging P(assisted). Show the number of analysed moves next to every value; Regan's ~1,500 moves is a reasonable "enough data" mark.
9. **Clean current-era negatives.** EGF and AGA in-person games are rarely published as SGF. Use the OGS `sgfs-uploaded` tournament set and GoGoD/Go4Go pro games dated 2019+ as the current-era null, and flag the pro null as "pros match engines ~60–70 %" rather than as amateurs' ceiling.
10. **Pitfalls to add.** OGS SGF ranks are wrong (use the JSON). Fox entry ranks are capped at 3d, and Fox 9d contains AIs. The KGS archive stops at 2019-04. The OGS rank scale changed on 2021-01-28 and drifts after 2024. There is no labelled cheater set, so detection rates are synthetic only, and the UI should say so.

## Correction (2026-11-23, checked against the files)

The za3k OGS dumps (the 2021 release's `sample-100k.json.gz` and the first records of
`ogs_games_2013_to_2025-05.json.gz`) contain only the game record: `players.{black,white}.rank` (a float at game
time, 30 = 1d), `ranked`, `handicap`, `time_control.speed`, `outcome`, and `moves` as `[x, y, ms]`. They do **not**
contain `historical_ratings`, rating deviation, `ui_class` or annulment fields; those come from the live API
(`/api/v1/games/{id}`), which the "R2 filters" above assumed. In the dumps, provisional ranks and bots have to be
found another way: name lists, the busy-account rule used on KGS, and if needed a single, rate-limited lookup of the
accounts that remain suspicious.
