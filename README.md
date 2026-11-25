# GoRanks — results and model data

Go rank estimation and AI-usage ("engine-likeness") analysis, built with [Gifu](https://github.com/JJLDonley/Gifu)
(the code lives in Gifu's `GoRanks/` folder). This repository holds the published results:

- **`site/`** — the viewer, published at https://jjldonley.github.io/goranks/ (rank bands, ranks, games; per-game
  boards, graphs and positional play). "How the data is made" on the site explains every column.
- **`data/vast/`** — training rows per batch of 20 games (KataGo 1,000 visits, KataGo-Human profiles at 100 visits).
- **`data/leela/`** — Leela 0.11 analysis (1,000 playouts) per game: the final candidate list for every position.
- **`data/sgf/`** — the analysed amateur game records (public OGS and KGS games).
- **`reports/`, `charts/`** — the training log, methods and summary charts.

Not published: the raw engine output, the games and accounts flagged as suspected AI use, professional game records.

Game records: OGS games from the OGS data dump (made with OGS's permission); KGS games from the u-go.net KGS archive.
