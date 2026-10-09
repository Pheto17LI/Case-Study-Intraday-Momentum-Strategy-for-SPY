# SPY Intraday Momentum: Maroy Parameter Selection

This research script compares the SPY intraday momentum strategies from the paper-style baseline with three approaches for adapting Maroy VWAP parameters over time: monthly validation, a two-state hidden Markov model (HMM), and a walk-forward Random Forest classifier. It also reports the M9 BiLSTM quantile strategy and M12 inverse-variance sizing strategy alongside the original strategies.

The Maroy methods are run as independent portfolios using the selected parameter profile for each date. They do not splice returns from other strategy equity curves.

## Strategies

- **Paper baselines:** opposite-band 1x, current-band plus VWAP 1x, and current-band plus VWAP with dynamic volatility targeting.
- **Maroy published parameters:** the published VWAP #1 settings, originally optimized on QQQ with 1-second data, applied here to SPY 1-minute data.
- **OWN v2:** adaptive noise area with adaptive stop; its regime threshold is selected using the 2021 validation year.
- **M9 BiLSTM q10/q90:** a bidirectional LSTM that estimates intraday lower and upper return quantiles. It uses the previous 20 sessions as a sequence, trains on up to 504 prior sessions, and refreshes every 504 sessions.
- **M12 inverse variance:** retains the current-band/VWAP signal and scales leverage using squared inverse trailing volatility, capped by the configured maximum leverage.
- **Maroy monthly validation:** selects the candidate profile with the strongest Sharpe ratio in the immediately preceding calendar month, then deploys it for the next month. If that month has too few valid sessions, it falls back to the published profile.
- **Maroy HMM regime selection:** refits a two-state diagonal Gaussian HMM monthly using trailing volatility, overnight-gap, and available VIX features. A causal filtered state determines which profile to use.
- **Maroy ML-predicted parameters:** a Random Forest classifier learns from completed prior months. Its label is the candidate profile with the best realized monthly Sharpe. Features include trailing realized volatility, returns, overnight gaps, and available VIX values. It only uses completed months; when there are fewer than 12 training labels or inadequate class diversity, it uses the monthly-validation selection as a fallback.
- **SPY buy and hold:** benchmark.

The candidate parameter profiles vary the Noise Area lookback, entry multiplier, target volatility, and intraday sampling frequency. The exact profiles and strategy settings are defined near the top of the Python file.

## Research design and assumptions

- **Asset/data:** SPY regular-session 1-minute bars and daily bars from Alpaca SIP, January 2016 through October 3, 2026; cash dividends are also loaded.
- **Costs:** commission is `$0.0035` per share with a `$0.35` minimum per order; baseline slippage is `$0.001` per share.
- **Chronological split:** train 2016–2020, validation 2021, and test 2022–2026. The fixed split is used in the reporting and OWN v2 threshold selection; the monthly Maroy methods use their own past-only rolling selections.
- **Optional VIX:** the script looks for a local VIX history file in `~/Downloads` (for example, `VIX_History.csv`). VIX features are used only when a compatible file is found.
- **Execution model:** signals are calculated from historical bars and costs are deducted in the backtest. This is a research simulation, not live order execution.

The published Maroy result is a QQQ/1-second benchmark. Applying its parameters and related selection methods to SPY/1-minute data is an adaptation, so results are not directly equivalent to the paper’s reported performance.

## Requirements

Use Python 3.10 or 3.11 and install the packages below in a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
python -m pip install --upgrade pip
python -m pip install requests pandas numpy pytz matplotlib statsmodels torch scikit-learn hmmlearn openpyxl
```

PyTorch installation instructions can vary by operating system and hardware. If the standard `pip install torch` command does not work for your system, use the installation command recommended by the official PyTorch site.

## Alpaca credentials and running

The script reads credentials from environment variables. Set them in your shell; do not place real credentials in the code or commit them to GitHub.

```bash
export ALPACA_API_KEY="your_key"
export ALPACA_SECRET_KEY="your_secret"
python spy-maroy-monthly-machine-learning-parameter-selection.py
```

PowerShell equivalent:

```powershell
$env:ALPACA_API_KEY="your_key"
$env:ALPACA_SECRET_KEY="your_secret"
python .\spy-maroy-monthly-machine-learning-parameter-selection.py
```

The script first checks for compatible cached minute, daily, and dividend CSVs in its output/cache locations and selected prior SPY output folders. If no complete cache is available, it requests the data from Alpaca and stores it in the output cache. The script requires the environment variables even when cached data is available.

## Outputs

By default, outputs are written to:

```text
~/Downloads/SPY_Paper_Replication_Maroy_ML_Parameters_Output/
```

The folder contains `figures/`, `tables/`, and `data/`, plus an Excel report, a README of run settings, a manifest, and a ZIP archive. Exports include the equity-curve comparison; portfolio and trade summaries; annual and monthly returns; train/validation/test results; parameter-selection logs; HMM state/profile assignments; daily audit data; and sensitivity/regime analyses.

## Reproducibility notes

- The script uses fixed random seeds for the BiLSTM, Random Forest, and HMM components where configured.
- The first months of the walk-forward models necessarily use their documented fallback because there is not yet enough completed history.
- Strategy settings are defined in the script. Changing them creates a different experiment; preserve the original settings when reproducing the archived output.
- Results depend on the Alpaca data snapshot, local VIX file, package versions, and platform.
