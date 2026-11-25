# Measure Go style as rank-matched residuals

Gifu should estimate a player's style as **the systematic difference between the move they chose and what two reference players would have chosen in the same position**: KataGo (the "ideal" baseline) and the KataGo human SL network at the player's own rank (the "peer" baseline). It should not use raw frequencies. Every field that measures style with any rigour, including chess (Regan's intrinsic ratings, Maia, behavioural stylometry), backgammon (signed cube errors), mahjong (rank-tier rate bands) and poker (opportunity-based HUD stats), reaches the same design: separate skill from preference, compare like positions with like, count opportunities rather than moves, and shrink small samples toward a population prior. Go has the best ingredients of any game for this. KataGo exposes win rate and score separately per candidate, ownership and its volatility, and a rank- and era-conditioned human policy, so most of the user's ideas (heat map, safe vs greedy, shape, human-likeness at rank, aggression, "bleeding") become computable residuals with explicit formulas. The evidence base is thin, though. **The only expert-validated Go style model predicted four 1–10 expert scales with RMSE ≈ 1.6 against ≈ 2.2 for guessing the mean**, territory and aggressiveness were learnable, "thickness" was not, and **no published Go player-identification or test-retest reliability figures exist**. Gifu therefore has to ship each dimension with a sample count, a shrunken estimate and an interval, and has to run its own reliability and validation study before it presents any "you are a fighter" label as fact. A first profile needs about 20 even, non-blitz games, and a stable one needs about 50.

## Every game that measures style well separates it from skill first

The strongest lesson from other games is that naive style metrics are mostly strength metrics in disguise. In chess, average error falls monotonically with rating (Regan's expected error is .0917 at Elo 2200 and .0561 at 2700). Regan also showed that Guid & Bratko's champion rankings were flawed because each player was scored on a different set of positions, with no normalisation "to a common reference set" ([Regan & Biswas 2013](https://cse.buffalo.edu/faculty/regan/papers/pdf/ReBiNF13.pdf)). Capablanca topped raw accuracy partly because he steered into simpler positions, and once complexity was controlled the order changed ([ChessBase on Guid & Bratko](https://en.chessbase.com/post/computers-choose-who-was-the-strongest-player-)). Regan's fix was to model each move choice as `p_i = p_0^α` with `α = exp((δ_i/s)^c)`. Here δ_i is the log-scaled value gap to the best move, so errors count less in already decided positions, s is "sensitivity" and c is "consistency" ([Regan & Biswas 2013](https://cse.buffalo.edu/faculty/regan/papers/pdf/ReBiNF13.pdf)). Two players of equal Elo can differ in s versus c ("precise but occasionally blundering" versus "steady but imprecise"), which makes the s/c balance itself a style axis that is orthogonal to strength by construction.

The machine-learning lineage arrives at the same place. Maia trains a human move predictor per rating band, and its accuracy peaks at the band it was trained on ([Maia](https://ar5iv.labs.arxiv.org/html/2006.01855)). Maia-2 conditions one model on skill directly ([arXiv 2409.20553](https://arxiv.org/abs/2409.20553v1)). Fine-tuning a personal model adds **about 4–5 percentage points of move-prediction accuracy over the rating-level model**, and that gain is the measurable "style beyond skill" ([McIlroy-Young et al. 2020](https://arxiv.org/pdf/2008.10086)). A 2026 paper builds Elo-disentangled style embeddings as a rating-conditioned residual on top of a base policy and reuses the architecture for Go with KataGo as the base ([arXiv 2606.25176](https://arxiv.org/pdf/2606.25176)). KataGo's human SL net (`rank_20k`…`rank_9d`, `preaz_*`, `proyear_1800`…`proyear_2023`) is exactly this kind of skill- and era-conditioned base policy for Go ([KataGo Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)). Gifu already has it.

The other games each add a piece.

**Backgammon: errors have a direction.** Engines split errors by direction as well as size. Missed doubles and wrong passes mark a conservative player; wrong doubles and wrong takes mark a bold one. PR is mean equity lost per *non-obvious* decision × 500 ([eXtreme Gammon](https://en.wikipedia.org/wiki/EXtreme_Gammon); [GNU BG manual](https://www.gnu.org/software/gnubg/manual/html_node/Checker-play-statistics.html)). The bold/timid ratio is a far better style index than total error.

**Mahjong: skill and style get separate panels.** Mahjong reads style from a small vector of rates (call, riichi and deal-in rates) against tier-specific bands. Deal-in of 10–11% reads as defensive and 12–13% as average ([reachmahjong](https://reachmahjong.com/en/forum/viewtopic.php?p=58576)). Engine agreement and bad-move percentage are kept on a separate skill panel, and NAGA even offers an "aggressive" model "with a similar level of skill" as a style reference ([Riichi Notes](https://riichinotes.blogspot.com/2023/06/reviewing-my-first-50-houou-games-with.html)).

**Poker: every stat has an opportunity count.** Each stat is a numerator over an explicit opportunity set, and the sample size is that count, not hands played. One pro "had only 11 opportunities to c-bet in the 300 hands" they played together ([PokerNews](https://www.pokernews.com/strategy/teachable-moments-from-twitch-ryan-laplante-hud-stats-32348.htm)). The aggression factor is treated as undefined below ten calls ([pokerskill](https://www.pokerskill.com/poker-glossary/aggression-factor/)).

**Shogi and chess: openings dominate identification.** Shogi pros are harder to identify than amateurs because they share fashionable openings ([IPSJ 222009](https://ipsj.ixsq.nii.ac.jp/records/222009); unconfirmed details). In chess, an openings-only baseline identifies 93% of players when the opening is included ([McIlroy-Young et al. 2022](https://ar5iv.labs.arxiv.org/html/2208.01366)).

**Scrabble and Othello: two values can trade off.** Scrabble's equity = score + rack leave ([Macondo](https://domino14.github.io/macondo/howitworks.html)) and Othello's mobility versus disc count ([Computer Othello](https://en.wikipedia.org/wiki/Computer_Othello)) both separate immediate gain from future potential. KataGo's scoreLead and winrate already make the same split.

**Player-type research: continuous axes beat discrete types.** Bartle's types did not survive factor analysis; the motivations co-occur rather than exclude each other ([Yee](https://nickyee.com/facets/revisiting.html)). BrainHex "cannot be reliably used to classify player preferences" ([Tondello et al.](https://hcigames.com/content/files/2024/07/A-Trait-Model-and-Scale-of-Game-Playing-Preferences.pdf)). Behavioural clusters in game telemetry often have silhouettes around 0.15–0.2 ([RPubs example](https://rstudio-pubs-static.s3.amazonaws.com/1315087_1c75cc6c371c4f74bd81d48b91b640da.html)).

Gifu should therefore report continuous, possibly correlated axes and treat archetype names such as "fighter" only as optional labels with a membership strength.

## Ten dimensions, each a residual against two baselines

The Go vocabulary supplies the axes. Across Japanese, Chinese and Korean writing, the most stable contrast is territory (実利) against influence/moyo (模様), followed by calm against fighting, thick/honte against light/fast, and orthodox against novel ([3rd-in pedia 棋風](https://pedia.3rd-in.co.jp/wiki/%E6%A3%8B%E9%A2%A8); [SL: Fast](https://senseis.xmp.net/?Pace); [SL: Honte](https://senseis.xmp.net/?Honte)). The GoStyle questionnaire turned these into four 1–10 scales: Territoriality (moyo↔territory), Orthodoxity (classic↔novel), Aggressivity (calm↔fighting) and Thickness (safe↔shinogi) ([Moudřík & Baudiš 2012](https://ar5iv.labs.arxiv.org/html/1209.5251)). Territory and thinness are separate axes. Cho Chikun is territorial and thin, Lee Changho territorial and thick, Takemiya influence-oriented and thick, so the estimator must not collapse the two. Endgame accuracy, reading and counting are skills, not styles, even though pro biographies mix them freely.

### The common machinery

**Notation.** Let *P* be the position before a move by player X, with candidate set *C*. Each candidate c has a KataGo search share w_K(c), win rate W(c), score S(c), scoreStdev σ(c), and, where requested, an ownership map O_c. Let h_r(c) be the human SL policy at rank r.

**Two baselines.** For any per-move feature f:

- KataGo baseline: `E_K[f] = Σ_c w_K(c)·f(c)`
- Peer baseline: `E_H,r[f] = Σ_c h_r(c)·f(c) / Σ_c h_r(c)`

Here r is X's estimated (or declared) rank. Rank and era profiles use 1-visit queries, since one visit "at full temperature" is the documented match to a profile ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)).

**The residual.** The per-move residual is `f(played) − E[f]`. The player-level score is the mean residual over the move's *opportunity set*. Gifu's existing height, contact, tenuki and zone comparisons already compute `f(played) − E_K[f]` in [src/core/style.h](/mnt/c/CataGui/src/core/style.h). The missing pieces are the peer baseline, the opportunity filters and the uncertainty.

**Two reasons for two baselines.** E_K answers "how unlike perfect play", which is confounded with strength. E_H,r answers "how unlike your peers", which is the style signal. Where the two disagree the gap is informative. A player can be more territorial than KataGo simply because every 5k is, and E_H,5k removes that.

**Opportunity filter (applies to every dimension).** Count a move only when it is a real choice:

- **Not forced.** The top KataGo share is below about 0.8, or at least two candidates are within ε = 1.5 points. This is the backgammon "non-obvious decisions" rule.
- **Not decided.** W is between 0.05 and 0.95, Regan's "errors matter less when decided". Play when far ahead is reported separately, under bleeding.
- **Outside the opening.** By default, from move 30 onward, plus a separate opening panel. The chess stylometry work skipped 15 plies to stop repertoire swamping the signal ([McIlroy-Young et al. 2022](https://ar5iv.labs.arxiv.org/html/2208.01366)).

**The near-equal set.** Several dimensions are cleanest when restricted to the near-equal set N(P), the candidates within ε points or δ = 3% win rate of the best. Inside N(P) there is no "right answer", so the choice there is pure preference. This is Regan's "style lives in which of the near-equal moves you pick", made explicit.

### The dimensions

| # | Dimension (poles) | Per-move formula | Opportunity set | Baseline that matters | Notes |
|---|---|---|---|---|---|
| 1 | Territory ↔ influence | line(m) ≥ 4, zone(m); with ownership: ΔT = Σ_p max(0, s·O_m(p))·[s·O_m(p) > 0.7] − same before (secured own points) vs ΔI = Σ_p s·O_m(p)·[0.2 < s·O_m(p) ≤ 0.7] (soft own area) | Non-forced moves, phased (1–40, 41–150, 151+) | E_H,r first, E_K second | Line × phase was the second-best GoStyle feature (RMSE 1.723 vs 2.168) ([arXiv 1512.08969](https://arxiv.org/pdf/1512.08969.pdf)); KaTrain's settledness is the code-defined version ([KaTrain ai.py](https://raw.githubusercontent.com/sanderland/katrain/master/katrain/core/ai.py)) |
| 2 | Calm ↔ fighting | contact(m); mean ownershipStdev over the board after m; σ(m) − E[σ] | Non-forced middle-game moves | E_H,r | ownershipStdev is KataGo's per-point "unsettledness" ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)) |
| 3 | Simplifying ↔ complicating | Inside N(P): σ(m) − mean σ over N(P); entropy of the next position's visit shares minus the current one | Positions where |N(P)| ≥ 2 | None needed (choice among equals) | Position-selection style à la Guid & Bratko |
| 4 | Local ↔ tenuki (tempo) | far(m, last) (existing); ω-local sente/gote sequence counts (ω = 5) | Answers to an opponent board move | E_K and E_H,r | GoStyle's ω-local feature was weak alone (RMSE 2.115) ([arXiv 1512.08969](https://arxiv.org/pdf/1512.08969.pdf)) |
| 5 | Safe (win rate) ↔ greedy (score) | See the safe-versus-greedy subsection below | Non-best moves with a known loss | Sign of the error, not its size | Backgammon signed-error idea |
| 6 | Solid ↔ thin (shape) | Pattern detectors (empty triangle, dumpling, …) and log h_9d(m) − log h_r(m) on contact moves | Local/contact moves | E_H,r | "Thickness" failed expert validation; present as shape, not thickness |
| 7 | Orthodox ↔ novel | −log h_r(m) residual against E_H,r[−log h_r]; era fit argmax_y Σ log h_proyear_y(m) | Non-forced moves | E_H,r | GoStyle's orthodoxity was learnable (1.773 vs 2.423) |
| 8 | Steady ↔ volatile accuracy | Fit (s, c) or report median loss and blunder rate (loss > 3 pts) separately | Non-forced, undecided | Same-rank humans | Regan's two-parameter split |
| 9 | Fast ↔ deliberate | t(m)/median_t(X), and corr(t, difficulty) with difficulty = human-weighted score spread at rank r | Moves with times, by time control | Player's own median | Time is used by Chessiverse and the 2026 signature paper ([arXiv 2606.18544](https://arxiv.org/abs/2606.18544)) |
| 10 | Human-likeness / rank fit | argmax_r Σ log h_r(m); per-move z-score against the rank-r distribution | All non-forced moves | All rank profiles | Displayed on the skill panel, not as style |

### Formulas that need more than the table

**Position difficulty.** Lightvector's Human SL guide defines the mean and spread of a position's outcome under human play at rank r:

- Human-weighted expected score: `m = Σ S(c)·h_r(c) / Σ h_r(c)`
- Its spread: `sqrt(Σ (S(c) − m)²·h_r(c) / Σ h_r(c))`

He also suggests weighting mistakes by how strongly a higher-ranked profile avoids them ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)). The spread is Gifu's position-difficulty measure. It underpins dimensions 8 and 9 and the aggression formula below. It is also the cheapest analogue of Biswas & Regan's complexity κ, which in chess needs rankings at several search depths ([Biswas & Regan 2015](https://scitepress.org/PublishedPapers/2015/52883/pdf/index.html)).

**Hard-to-find moves.** A move with low prior but a high final visit share is the Go analogue of Biswas–Regan "swing up". Choosing such moves more often than same-rank humans do marks a calculating or creative player.

**The ownership-based territory measure.** This is the most promising upgrade to Gifu's line-height measure, but it is the least proven. The only prior art is a 2023 IPSJ paper that ran PCA on KataGo ownership histograms and reportedly found axes for stone and territory uncertainty. Only its abstract was seen ([IPSJ 229344](https://ipsj.ixsq.nii.ac.jp/records/229344)).

**Cost of ownership baselines.** The played move's ownership is free, because it is the next position's root ownership. The baseline E_K over alternatives needs `includeMovesOwnership` or extra queries. A practical compromise is to compare the played move's ΔT/ΔI with the top KataGo candidate's only.

## Turning the user's six ideas into numbers that survive scrutiny

### Heat map

The heat map is a presentation layer, not a dimension, and on its own it mostly shows the game rather than the player. Gifu already stores a 19×19 heat grid per player. Three changes make it a style picture:

- **Difference map.** Show the player's heat minus KataGo's expected heat over the same positions (Σ w_K at each point). Also show the player's heat minus the human-rank-r expected heat.
- **Symmetry folding.** Fold the board by its 8 symmetries into one octant, so that "plays the 3-4 point" does not split four ways.
- **Phase split.** Split by phase, since an opening heat map is repertoire ([McIlroy-Young et al. 2022](https://ar5iv.labs.arxiv.org/html/2208.01366)).

The difference map's marginals by line and zone are dimension 1, which keeps the picture and the number consistent.

### Safe versus greedy

Dimension 5 is the user's win-rate-versus-score idea, built on backgammon's signed errors.

**Per-move greed.** For each non-best move m, let b_W be KataGo's win-rate-best candidate and b_S its score-best candidate. Define two losses:

- ΔW = W(b_W) − W(m)
- ΔS = S(b_S) − S(m)

The move is **greedy** if S(m) > S(b_W) while W(m) < W(b_W): the player bought points at the cost of win probability. It is **safe** if S(m) < S(b_W) while W(m) ≥ W(b_W) − δ: the player gave up points to keep the win rate.

**Player-level greed index.** `G = (greedy − safe) / (greedy + safe)`, shrunk toward the rank prior.

**A second safety signal.** Use Lightvector's z-score idea on σ: among near-equal candidates, did the player pick lower-scoreStdev moves when ahead and higher ones when behind? That is the mahjong "playstyle changes with the lead" analysis ([Path of Houou](https://pathofhouou.blogspot.com/2021/02/analysis-playstyle-changes-based-on-lead.html)) and backgammon's "trailer should be more aggressive" ([match strategy](https://wiki.riteme.site/wiki/Backgammon_match_strategy)). Splitting G by W bucket (behind < 0.4, even, ahead > 0.6) separates **risk appetite** from **risk adaptivity**, which are two different traits.

### Good and bad shapes

These need two complementary detectors.

**Explicit patterns.** Recognise named patterns on the board: empty triangle, dumpling, hane at the head, and so on. Count them per *local-move opportunity* and residualise against how often the rank-r human policy would play the same pattern there. Explicit patterns are what players understand and accept as explanations.

**Policy shape.** On contact and local moves, compute `log h_9d(m) − log h_r(m)`. A strongly negative value means "a 9d would almost never play this here, though your peers might". This is rank-robust and catches shapes no pattern list names. The guide's own release notes show the effect: the 20k profile simply follows its opponent, while 1d favours tiger's-mouth shape ([KataGo v1.15.0](https://github.com/lightvector/KataGo/releases/tag/v1.15.0)).

**Label it shape, not thickness.** GoStyle's thickness scale was **barely better than the mean (RMSE 1.567 vs 1.672)** because the experts could not separate players on it ([arXiv 1512.08969](https://arxiv.org/pdf/1512.08969.pdf)). Gifu should label this axis "shape" and should not claim to measure "thickness".

### Human-play estimate at the player's rank

**Rank fit.** The rank estimate is a likelihood fit: `r̂ = argmax_r Σ_moves log h_r(m)` over non-forced moves, with a profile-likelihood interval. The same fit over `proyear_*` gives "your moves look most like pros of year Y". This is an inference built on the human SL design, not Lightvector's published recipe.

**Accuracy limits.** Pre-deep-learning Go rank prediction achieved about 2.4 ranks σ from 85 games and 4.6 ranks from 9 games ([Baudiš & Moudřík 2012](https://ar5iv.labs.arxiv.org/html/1209.5251)). The human net should do better, but this is unmeasured. KataGo warns that the training data includes sandbagging and AI-cheating games, and that the net is less accurate at high dan ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md); [v1.15.0](https://github.com/lightvector/KataGo/releases/tag/v1.15.0)).

**Rank anchors the peer baseline.** r̂ is the r used in every E_H,r. When the SGF declares a rank, use the declared rank for the baseline and show r̂ beside it. Otherwise a 5k who plays like a 2k would be compared with 2ks.

**Distinctiveness.** The "personalisation gain" diagnostic tells the player how distinctive they are: how much better a player-specific mixture predicts their moves than h_r alone. In chess this gain is about 4–5 points ([arXiv 2008.10086](https://arxiv.org/pdf/2008.10086)). When the gain is near zero, the honest message is "you play like a typical r̂".

### Aggression

The user defines aggression as playing fast, plus bad shapes that pressure the opponent into blunders, plus locally bad exchanges for a global move. These should be three measured components, and the composite should never be shown without them. Naive aggression proxies contradict reputation: a "squares attacked" metric ranked Tal near the bottom of chess's attackers ([Berkeley SAG](https://sportsanalytics.studentorg.berkeley.edu/articles/attacking-in-chess.html)).

**(a) Speed.** The time residual from dimension 9, restricted to fighting positions: positions whose difficulty spread is above the player's median. Fast play in easy positions is not aggression.

**(b) Pressure.** For each player move m, let D_opp(P′) be the human-weighted expected loss at the *opponent's* rank in the next position P′: `D_opp = Σ_c h_r_opp(c)·(S_best − S(c)) / Σ h`.

- **Pressure** of m: `D_opp(P′ after m) − D_opp(P′ after KataGo's top move)`. This costs one extra 1-visit human query plus a shallow evaluation per move.
- **Aggressive bad shape:** moves with positive pressure that also have negative policy shape (dimension 6) and a non-zero point loss. These are deliberate overplays.
- **Pressure success:** compare the opponent's actual loss on the reply with D_opp. This is an *outcome* statistic, like baseball's BABIP, so it will stabilise slowly ([FanGraphs](https://library.fangraphs.com/principles/sample-size/)). Show it as "your pressure paid off X points per game", with a wide interval.

**(c) Local-for-global.** A move is a **local concession** when the ownership swing inside radius R = 4 of the most recent fight is against the player by ≥ L points, while the move is far from that fight and its total score loss is < 1 point. This is KataGo-style tenuki or sacrifice.

- Count these per *opportunity*: positions where KataGo's local reply is not forced.
- Compare the rate with E_H,r. Its "sign" against KataGo tells whether the player's trades are good, but the style is the willingness to make them.

**The composite.** Show the three z-scored components side by side, and a composite only as their mean once each has reliability ≥ 0.5. This matters because (b) and (c) can disagree: Lee Sedol's "risk lover" style pairs naturally with (b), Cho Hunhyun's "fighting jabs" and speed with (a) ([SL: Lee Sedol](https://senseis.xmp.net/?LeeSedol)).

### Bleeding

Bleeding needs a window definition and a direction test, or it will just re-measure endgame skill.

**Windowed bleed.** Take sliding windows of W = 30 of the player's moves, with an endgame variant from move 150 on. Within each window, sum only the *small* score losses: moves with 0 < loss < 2 points, below KaTrain's 3-point mistake bucket ([KaTrain config](https://raw.githubusercontent.com/sanderland/katrain/master/katrain/config.json)).

- B = Σ loss_i.
- A window **bleeds** if B ≥ 5 points and no single move in it lost ≥ 2.

This captures the user's "5 points over 30–40 moves" exactly.

**Peer-adjusted bleed.** `B_H = Σ (loss_i − E_H,r[loss])`, where E_H,r[loss] is Lightvector's human-weighted expected loss in that position at rank r. A 10k who bleeds 5 points where typical 10ks bleed 6 is not "too safe"; a 3d who bleeds 5 where 3ds bleed 1.5 is.

**Direction.** Classify each bleeding move as one of three kinds:

- **Too safe:** the move had lower σ than the best candidate, or it is a gote defensive reply or a reinforcement inside own territory where KataGo tenukis.
- **Too slow:** a correct-area move that is gote where a sente one existed.
- **Careless:** neither.

**Rational slack.** KataGo itself sheds points when far ahead, because it maximises win probability. A player at W > 0.9 who gives away points while W stays flat is playing rationally. That is a separate "closes out won games" trait, worth reporting but not as a flaw. Kobayashi Koichi's "unmatched skill at winning a won game" is this trait in a pro ([SL: Professional Players' Go Styles](https://senseis.xmp.net/?ProfessionalPlayersGoStyles)).

**Display.** Report the share of windows that bleed, the mean B_H, and the too-safe fraction.

## Twenty games for a provisional profile, fifty for a stable one, always with intervals

No Go style statistic has a published reliability curve, so Gifu must measure its own. The evidence from other games gives the shape and the scale.

**Reliability is a curve, not a threshold.** Baseball's convention calls a statistic "stable" when split-half r reaches 0.7. Statistics the player *chooses* get there fast (strikeout rate in about 60 plate appearances). Outcome-dependent ones are slow (BABIP needs 820–2000 balls in play) ([FanGraphs Library](https://library.fangraphs.com/principles/sample-size/)). Go's per-move choice metrics (line, contact, tenuki, near-equal choices, greed sign) should behave like strikeout rate. Pressure success, captures and score at move 150 will behave like BABIP.

**Chess and commercial tools.** Chess stylometry "starts to saturate around 50 games" per side and is "reasonable" with 10 ([arXiv 2208.01366](https://ar5iv.labs.arxiv.org/html/2208.01366)). Chessiverse requires 20 games, "ideally 50+", and admits 20–30-game profiles "may shift" ([Chessiverse](https://chessiverse.com/chess-personality)).

**Go engine-error metrics are noisier.** A backgammon match of about 230 decisions gives only ±2.5–3.5 PR, and "we generally need more data than we think" ([Madsen](https://lassehjorthmadsen.github.io/bganalyses/analyses/pr-confidence-intervals.html)). Regan's intrinsic rating from 680 moves still carries ±170 Elo at 2σ, and its error bars are inflated ×1.15–1.4 because moves within a game are not independent ([Regan & Biswas 2013](https://cse.buffalo.edu/faculty/regan/papers/pdf/ReBiNF13.pdf)).

**GoStyle's precedent.** GoStyle used 16-game samples and still reached only about ±1.6 on a 10-point scale ([arXiv 1512.08969](https://arxiv.org/pdf/1512.08969.pdf)).

**Gifu's working thresholds.** "Provisional" from 20 qualifying games; "stable" when every displayed axis has reliability ≥ 0.7. Expect that to take about 50 games for loss-based and outcome axes, and fewer for choice axes.

**One machinery for shrinkage, intervals and "how many games".** Fit reliability as `r(n) = n/(n+k)` from permutation split-half data, splitting by game, never by move or by first and second half of a game.

- **Shrinkage.** Display the shrunken estimate `(n·x̄ + k·μ_rank)/(n + k)`. This is Tango's "add k average trials" ([Tango](https://tangotiger.net/archives/stud0322.shtml)), with μ_rank the mean for the player's rank band.
- **Required games.** k is also the number of games at which the player's own data gets half the weight. Spearman–Brown, `n = ρ*(1−ρ)/(ρ(1−ρ*))`, converts any measured reliability into games needed for a target ([Spearman–Brown](https://en.wikipedia.org/wiki/Spearman%E2%80%93Brown_prediction_formula)).
- **Intervals.** Bootstrap over *games* (resample whole games) for 90% intervals. This automatically includes within-game correlation, which per-move binomial errors understate.
- **Opportunity counts.** Show n opportunities beside every stat, as poker HUDs do. Grey out any axis with fewer than about 30 opportunities or reliability below 0.5, the equivalent of the poker rule that the aggression factor is undefined below ten calls ([pokerskill](https://www.pokerskill.com/poker-glossary/aggression-factor/)).

**Presentation.** Show each axis as a bar holding the shrunken value, its 90% interval, and a marker for the rank-band median. Show a percentile within rank band and era rather than a raw z-score, because skewed metrics such as blunder rate break normality.

**Multiple comparisons.** With ten axes at 90% intervals, about one axis will look extreme for a perfectly typical player. Highlight a "signature trait" only if it survives shrinkage *and* its interval excludes the middle half of the rank band.

**Single games.** A single game is shown as "this game against your usual" (deviation from the shrunken profile), never as "your style". Mahjong players already track style drift in roughly 100-game windows ([kyuual](https://kyuual.blogspot.com/2020/04/playstyle-every-100-games-or-so.html)), so Gifu should show a rolling profile alongside the all-time one.

## Validation must beat an opening-only baseline and the GoStyle table

Gifu should validate in five layers, roughly from cheapest to most persuasive.

**1. Split-half reliability per axis.** Collect players with at least 2N games (KGS, OGS or Fox archives, plus the user's own). Draw 100+ random disjoint game splits at N = 1, 2, 5, 10, 20 and 50, correlate across players, and fit k.

- Compute reliability *within* rank bands. Mixing 20k–9d inflates reliability, because talent spread drives the correlation ([FanGraphs](https://blogs.fangraphs.com/a-new-way-to-look-at-sample-size)).
- Expect lower between-player variance, and so lower reliability, among dans and pros. In chess, elite players were the hardest to identify (P@1 ≈ 0.30) ([arXiv 2208.01366](https://ar5iv.labs.arxiv.org/html/2208.01366)), and post-AlphaGo pro openings converge on AI choices. Shin Jinseo's moves matched AI 37.5% of the time against a 28.5% average ([MIT Technology Review](https://www.technologyreview.com/2026/02/27/1133624/ai-is-rewiring-how-the-worlds-best-go-players-think/amp/)).

**2. Player identification.** Build the style vector from reference games, then rank the true player among N candidates using held-out query games. Report top-1 and top-5 accuracy.

- Run it twice: with all moves, and with the first 30–50 moves removed.
- Compare against an opening-only baseline (first-move and joseki n-gram frequencies).
- If the baseline matches the style vector, the vector measures repertoire. This is exactly the trap chess fell into: a 93% baseline with openings included ([arXiv 2208.01366](https://ar5iv.labs.arxiv.org/html/2208.01366)).
- No Go figure exists, so any result Gifu gets is new.

**3. Expert labels.** Use the GoStyle table: 24 pros rated by 3 strong players on τ, ω, α, θ, with a reported answer standard error of 0.952 ([Baudiš & Moudřík 2012](https://ar5iv.labs.arxiv.org/html/1209.5251)). Compute Gifu's axes on each pro's peak-era GoGoD games and report Spearman ρ per axis.

- The labels are noisy and n ≈ 24, so treat this as a sanity check, not a fit target. A stacking follow-up already overfit this dataset ([arXiv 1512.09254](https://arxiv.org/pdf/1512.09254)).
- Required orderings:
  - **Territory:** Takemiya (τ 1.3) and Miyazawa (1.5) far below Cho Chikun and Kobayashi Koichi (both 9.0).
  - **Fighting:** Kato, Miyazawa, Yamashita (α 9.5) and Lee Sedol (9.3) far above Kobayashi Koichi (2.5) and Lee Changho (2.6).
  - **Thin/shinogi:** Cho Chikun (θ 9.0), Sakata and Ma Xiaochun (8.0) far above Takemiya (1.3) and Lee Changho (2.6).
- Expect territory to validate best and aggressiveness second, mirroring GoStyle's 1.52× and 1.41× improvements over the mean ([arXiv 1512.08969](https://arxiv.org/pdf/1512.08969.pdf)).
- Style drift gives extra checks. Kato Masao should show a drop in aggressiveness around 1990, when "The Killer" became "Endgame Kato". Rin Kaiho should move from territory toward influence ([SL list](https://senseis.xmp.net/?ProfessionalPlayersGoStyles)).

**4. Synthetic players of known style.** Generate games from KaTrain's style AIs: Territory, Influence, Local, Tenuki and Simple Ownership ([KaTrain ai.py](https://raw.githubusercontent.com/sanderland/katrain/master/katrain/core/ai.py)). Add human SL profiles at fixed ranks and years, and KataGo with score-versus-win-rate utility variants. Check that each axis recovers the generator's tendency (rank correlation, or ARI for types). This follows the MicroRTS protocol of scoring clusters against known agent identities ([arXiv 2507.10172](https://arxiv.org/pdf/2507.10172)) and the shogi practice of validating a style by self-play ([Omori & Kaneko](https://rd.springer.com/content/pdf/10.1007%2F978-3-319-42911-3_31.pdf)). It is the only validation with known ground truth.

**5. Discriminant validity and structure.**

- Correlate every axis with rank across the population. An axis with |r| > 0.5 is mostly skill and needs stronger rank conditioning; this threshold is a convention, not a sourced rule.
- Compute the player-level correlation matrix after shrinkage. Merge any pair with |r| > 0.7.
- Run PCA on one half of the players and confirm on the other. GoStyle's first pattern component was essentially territoriality (r ≈ −0.53) ([arXiv 1209.5251](https://ar5iv.labs.arxiv.org/html/1209.5251)), so a similar first component in Gifu's data would be convergent evidence.

## Five confounds that will masquerade as style

| Confound | How it fakes style | Mitigation in Gifu | Evidence |
|---|---|---|---|
| Strength | Weaker players look "fighting" (overplays) and "territorial" (low, slow moves); every engine-agreement metric tracks rank | Peer baseline E_H,r at declared or estimated rank; rank-band percentiles; style and skill on separate panels | Regan's AE falls monotonically with Elo ([Regan & Biswas](https://cse.buffalo.edu/faculty/regan/papers/pdf/ReBiNF13.pdf)); "power go is a personality defect" among amateurs ([L19](https://lifein19x19.com/viewtopic.php?p=70642)) |
| Opponent | Fight rate depends on whether the opponent fights; handicap forces White to overplay and lets Black play safe | Exclude or separate handicap games; record opponent rank and style; same-position residuals, not raw shares | FanGraphs reliability is "unadjusted for park, league, pitcher" ([FanGraphs](https://blogs.fangraphs.com/a-new-way-to-look-at-sample-size)); human SL is out of distribution for rank gaps above 9 or mismatched handicap ([Analysis_Engine.md](https://raw.githubusercontent.com/lightvector/KataGo/master/docs/Analysis_Engine.md)) |
| Time control | Blitz raises error and simplifies choices; it also inverts the "fast = aggressive" signal | Store the time control (SGF TM/OT); stratify; never pool blitz with long games; normalise times within time control | Fast chess drops intrinsic ratings by "several hundred Elo" ([Regan & Biswas](https://cse.buffalo.edu/faculty/regan/papers/pdf/ReBiNF13.pdf)); Maia drops moves with under 30 s left ([Maia](https://ar5iv.labs.arxiv.org/html/2006.01855)) |
| Phase and opening | Fuseki and joseki repertoire dominates identification and heat maps; endgame inflates "calm" and "local" | Default to move 30+, with a separate opening panel; phase-split every axis; odd/even game splits, never first and second half | Openings-only baseline reaches 93% in chess ([arXiv 2208.01366](https://ar5iv.labs.arxiv.org/html/2208.01366)) |
| Era | AI-era openings (early 3-3, fast development) make everyone look "modern" and engine-like; pre-2016 labels do not transfer | Era-matched reference populations; `preaz_` against `rank_` profiles; treat "similarity to KataGo" as its own post-AI axis | Opening divergence spiked in 2018, then returned to pre-AlphaGo levels ([Beheim 2025](https://pmc.ncbi.nlm.nih.gov/articles/PMC12516593/)); pro decision quality rose after AlphaGo ([Shin et al.](https://export.arxiv.org/abs/2303.07462)) |

Three smaller hazards deserve guards too.

**Selection bias.** Users upload wins, fights and review games, so Gifu should show which games each profile rests on and how many.

**AI-assisted or bot games.** Engine agreement that is implausibly high turns "style" into KataGo's style. Flag those games and exclude them.

**Engine-budget drift.** Visit shares and σ change with visits, and chess engines' tie-breaking biased which move counted as best (58/42) ([Biswas & Regan](https://scitepress.org/PublishedPapers/2015/52883/pdf/index.html)). Fix the visit budget for profile analysis, and record it with the cached evaluations.

## Implementation order for Gifu

The order runs from what the existing data supports today to what needs new engine queries. Every threshold (ε, δ, window W, bleed limits, opening cutoff, pattern list) belongs in a data file, for example `assets/data/style.kv`, covered by `tests/test_data.c` as the project conventions require.

**Step 1. Opportunities, phases and per-game storage** (`core/style.c`, `core/profile.c`).

- Add the forced, decided and opening filters and the three-phase split to the existing height, contact, tenuki and zone measures.
- Count opportunities explicitly.
- Store `StyleCounts` per game rather than only summed in `ProfileTotals`. Bootstrapping and split-half both need whole-game resampling, and today's totals cannot be resampled.

**Step 2. Win-rate versus score signed errors and the near-equal set.**

- Greed index G, by W bucket.
- Near-equal choice volatility (dimension 3).
- Steady-versus-volatile loss texture (median loss against blunder rate).

All of this uses per-candidate win rate, score and σ that Gifu already holds.

**Step 3. Bleeding windows.** Per-move score loss already exists. Add sliding-window B, the "bleeds" flag, the too-safe/slow/careless direction, and the "closing out won games" split. Show it first against E_K, then upgrade to B_H after step 4.

**Step 4. Peer baseline.**

- One 1-visit human SL query per analysed position, at the declared or estimated rank (Gifu already sends move history to the human model).
- Fit r̂ and the era fit.
- Compute E_H,r for all axes and the human-weighted difficulty spread.

This step turns every existing "vs KataGo" comparison into a "vs your peers" comparison, and it is the single biggest validity gain.

**Step 5. Shape.** Add explicit pattern detectors in `core` (no raylib) and the policy-shape residual log h_9d − log h_r on local moves.

**Step 6. Ownership-based territory and influence.** Compute ΔT and ΔI from the played move's next-position ownership and the top candidate's ownership, plus mean ownershipStdev for dimension 2.

**Step 7. Time features.** Parse SGF BL/WL/TM/OT into per-move times. Compute the time residual and the time–difficulty correlation, stratified by time control.

**Step 8. Aggression composite.** Add the opponent-rank human queries for D_opp after the played move and after KataGo's top move. Then pressure, aggressive bad shapes, local-for-global concessions, and a composite gated on reliability.

**Step 9. Reliability and validation harness.** A headless command over an SGF folder that does three things:

- Outputs per-game style vectors.
- Runs permutation split-half and fits k for each axis.
- Runs identification with and without the opening, against the opening n-gram baseline.

Then run it on GoGoD games for the GoStyle 24 and on synthetic KaTrain-style and human-SL games. Ship k values in the data file.

**Step 10. Presentation.**

- Shrunken bars with intervals and rank-band markers, greyed below reliability 0.5.
- Folded difference heat maps by phase.
- "This game against your usual".
- Archetype names only as optional nearest-prototype labels with a distance.

## Conclusion

The novel opportunity here is that Go, unlike chess, comes with an off-the-shelf, rank- and era-conditioned human policy and a split between win-rate and score value at every candidate. That lets Gifu define style the way the best chess work only approximates it: as the player's choices *among near-equal moves*, relative to *their own peers in the same position*. It also makes several of the user's intuitions sharply testable rather than impressionistic. "Too safe" becomes a signed error, "bleeding" a peer-adjusted windowed sum, and "aggression" a measurable rise in the opponent's expected error at the opponent's rank. The open risk is that nobody has yet shown any of these Go measures to be reliable or to agree with experts.

The validation harness is therefore not optional polish. It is the step that decides which axes Gifu is allowed to show with a label. The first result it produces, Go player-identification accuracy with the opening removed, would itself be a first for the field.
