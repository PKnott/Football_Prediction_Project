# Football Prediction Project

A two-stage data pipeline that pulls historical and live football statistics from public APIs, builds exponentially-decayed prior features, and predicts match outcomes using an XGBoost Poisson regression model. Outputs a formatted Excel workbook with scoreline probability matrices, over/under odds, and 1X2 probabilities for each fixture.

Covers the Premier League, La Liga, Bundesliga, Serie A, and Ligue 1.

---

## Key Results

All metrics evaluated on a held-out test set (Premier League matches after June 2024).

| Metric | Value |
|---|---|
| Poisson Log-Loss | 1.485 |
| MAE — predicted λ vs actual goals | 0.945 |
| Top-1 bucket accuracy (0–6+ goals) | 33.2% |
| Within ±1 bucket of actual | 82.0% |
| Overestimation rate (λ > actual) | 59.1% |
| Underestimation rate (λ < actual) | 40.9% |

The model has a slight tendency to overestimate goal rates, consistent with a Poisson model that doesn't account for low-scoring defensive setups. See `examples/` for real prediction outputs across all five leagues.

---

## Project Structure

```
Football_Prediction_Project/
├── Notebooks/
│   ├── Data Pull/
│   │   └── Data Pull.ipynb          # Data ingestion — run this first
│   └── Leagues/
│       ├── Prem_Predictions.ipynb
│       ├── Liga_Predictions.ipynb
│       ├── Bund_Predictions.ipynb
│       ├── Serie_Predictions.ipynb
│       └── Ligue_Predictions.ipynb
├── Inputs/                          # All raw data (git-ignored)
│   ├── Prem/
│   │   ├── Previous Seasons/
│   │   │   └── E0 Previous Seasons.csv
│   │   └── Current Season/
│   │       └── E0 Current Season.csv
│   ├── Liga/   Bund/   Serie/   Ligue/   (same structure)
│   ├── Fixtures/
│   │   ├── premier_league_fixtures.csv
│   │   ├── la_liga_fixtures.csv
│   │   ├── bundesliga_fixtures.csv
│   │   ├── serie_a_fixtures.csv
│   │   ├── ligue_1_fixtures.csv
│   │   ├── historic_competitions_fixtures.csv
│   │   └── all_competitions_fixtures_<season>.csv
│   ├── Mapping/
│   │   ├── Club_Mapping_<season>.csv
│   │   ├── Club_Mapping_All.csv
│   │   └── ESPN_Team_IDs.csv
│   └── soccerdata_cache/            # soccerdata HTTP cache (auto-managed)
├── Outputs/                         # Model outputs (git-ignored)
│   ├── Prem/
│   │   └── Prem_predictions_<dd_mm>.xlsx   # dated by run date
│   └── Liga/   Bund/   Serie/   Ligue/
├── examples/
│   ├── Prem_predictions_17_05.xlsx  # Real outputs from a test run
│   ├── liga_predictions_17_05.xlsx
│   ├── bund_predictions_17_05.xlsx
│   ├── serie_predictions_17_05.xlsx
│   └── ligue_predictions_17_05.xlsx
├── README.md
├── requirements.txt
├── .gitignore
└── LICENSE
```

---

## Notebooks

### 1. `Notebooks/Data Pull/Data Pull.ipynb` — Data ingestion

Pulls and structures all data into the `Inputs/` folder.

| Section | What it does |
|---|---|
| Previous Seasons | Fetches 10+ seasons of team match stats from Understat via `soccerdata`, saves to `Inputs/<League>/Previous Seasons/` |
| Current Season | Fetches the current season's stats, saves to `Inputs/<League>/Current Season/` |
| Fixtures | Pulls upcoming fixtures from ESPN, maps team names via fuzzy matching, saves to `Inputs/Fixtures/` |
| Past Fixtures (Comp Agnostic) | Fetches each team's full fixture history across all competitions, saves to `Inputs/Fixtures/` |

Run this notebook first, or whenever you need to refresh the data.

### 2–6. `Notebooks/Leagues/` — Model & output

One notebook per league. Each reads from `Inputs/`, trains the model, and writes predictions to `Outputs/<League>/`.

| Notebook | League | Reads from | Writes to |
|---|---|---|---|
| `Prem_Predictions.ipynb` | Premier League | `Inputs/Prem/` | `Outputs/Prem/` |
| `Liga_Predictions.ipynb` | La Liga | `Inputs/Liga/` | `Outputs/Liga/` |
| `Bund_Predictions.ipynb` | Bundesliga | `Inputs/Bund/` | `Outputs/Bund/` |
| `Serie_Predictions.ipynb` | Serie A | `Inputs/Serie/` | `Outputs/Serie/` |
| `Ligue_Predictions.ipynb` | Ligue 1 | `Inputs/Ligue/` | `Outputs/Ligue/` |

Each notebook follows the same pipeline:

| Section | What it does |
|---|---|
| Data Loading | Loads CSVs from `Inputs/<League>/` |
| Build Tables | Structures data into per-team per-match rows |
| Rest Days | Calculates days since each team's last match (any competition) |
| Promoted Teams | Flags newly promoted teams with no prior top-flight data |
| Strength Function | Exponentially decayed weighted mean over the last L matches |
| Priors | Computes pre-match attack/xG/defensive/points-per-game priors per team |
| Modelling Table | Assembles the feature matrix for XGBoost |
| Grid Search | Tunes decay rate and lookback window (L) per feature via Poisson log-loss |
| Final Model | Trains on best hyperparameters; evaluates on held-out test set |
| Diagnostics | Log-loss, MAE, bucket accuracy, over/under calibration |
| Output | Writes a formatted multi-sheet Excel file to `Outputs/<League>/` |

---

## Setup

**Requires Python 3.12.** Python 3.13+ is not fully supported by all dependencies (`soccerdata` in particular). If you have multiple Python versions installed, be explicit:

**macOS only — install `libomp` before creating the venv.** XGBoost requires OpenMP and it is not bundled on macOS:

```bash
brew install libomp
```

Then create the environment and install dependencies:

```bash
# From the project root
python3.12 -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Then open the notebooks in VS Code (or JupyterLab) and run from the project root so that relative paths resolve correctly.

### How paths work

Each notebook resolves the project root automatically — no manual configuration needed:

```python
PROJECT_ROOT = os.path.abspath(os.path.join(os.getcwd(), "..", ".."))
INPUTS  = os.path.join(PROJECT_ROOT, "Inputs")
OUTPUTS = os.path.join(PROJECT_ROOT, "Outputs")
```

This assumes the Jupyter kernel's working directory is the notebook's own folder, which is the default behaviour in VS Code. If using JupyterLab started from the project root, change `"..", ".."` to `"."`.

**Google Colab users:** commented-out setup instructions are provided at the top of each notebook.

---

## Day-to-Day vs. Start-of-Season Running

`Data Pull.ipynb` has sections that only need to run **once per season** and sections that need to run **before each prediction run**. Running everything every time is slow and unnecessary.

### Start of season (run once)

These cells are expensive and their output doesn't change until the next season starts:

| Cell | What it does | When to re-run |
|---|---|---|
| **Previous Seasons** | Fetches all historical match data | Start of next season |
| **Build ESPN→Understat Mapping** | Fuzzy-matches team names across sources | Start of next season (new promoted teams) |
| **ESPN Team IDs** | Resolves numeric IDs for every club | Start of next season |
| **Past Fixtures (All Competitions)** | Full cross-competition fixture history for rest-day feature | Start of next season |

### Before each prediction run (day to day)

These three cells take a minute each and give the model the latest information:

| Cell | What it does |
|---|---|
| **Current Season** | Refreshes this season's match stats from Understat |
| **Fixture Pull** | Pulls the upcoming fixtures for your chosen date range |
| **Current Season All-Comp Fixtures** | Updates rest-day data for the current season |

After running those three, open the relevant league notebook(s) and run top-to-bottom to get fresh predictions.

---

## Model Overview

- **Task:** Predict the expected goal rate (λ) per team per match, modelled as a Poisson process
- **Algorithm:** XGBoost with `objective=count:poisson`
- **Features:** Exponentially decayed priors for goals for, xG for, npxG against, and points per game (home/away split), plus rest days and promoted flag
- **Hyperparameter tuning:** Grid search over decay rate and lookback window (L) per feature, optimised by Poisson log-loss on a validation set
- **Evaluation:** Poisson log-loss on a held-out test set (matches after June 2024)
- **Output:** Full scoreline probability matrix + over/under + 1X2 + BTTS, formatted in Excel with colour-coded edge vs. league average

---

## Future Improvements

- **Player availability** — incorporate injury and suspension data to adjust team strength priors
- **Ensemble models** — combine XGBoost with a Dixon-Coles or neural Poisson model to reduce variance
- **Streamlit dashboard** — interactive UI to browse predictions, filter by league, and compare model odds vs. bookmaker lines
- **Automated daily predictions** — schedule the data pull and prediction notebooks to run overnight and push outputs to a shared location

---

## Dependencies

See `requirements.txt`. Requires Python 3.12.
