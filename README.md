# ai-trade-tracker

Interactive Bokeh application for visualizing U.S. imports of AI-related products. Data from the U.S. Census Bureau, classified using an LLM-based tool that maps HS10 commodity codes to AI data center infrastructure categories.

> **Citation:** Waugh, Michael E. "Trade in AI-Related Products." Working Paper, Federal Reserve Bank of Minneapolis, March 2026.

---

## Repository Structure

```
main.py                          # Bokeh application
prep_data.py                     # Data pipeline script
requirements.txt                 # Python dependencies (bokeh, pandas, pyarrow)
runtime.txt                      # Python version (3.12.8)
Procfile                         # Heroku deployment configuration
data/
    ai_trade_series.parquet      # Pre-computed output (produced by prep_data.py)
data-input/
    TOTALdata-current.parquet    # Raw U.S. Census Bureau monthly trade data
    hs10_classification_final_v3.csv  # HS10 code classifications (AI relevance + category)
```

---

## Data Pipeline

The pipeline is a single script (`prep_data.py`) that reads the raw trade data and classification file, computes monthly series, and writes a compact parquet file consumed by the app. Execute with `python prep_data.py`.

### Inputs

| File | Description |
|---|---|
| `data-input/TOTALdata-current.parquet` | Raw Census Bureau monthly import data. Key columns: `I_COMMODITY` (HS10 code), `time` (YYYY-MM), `CON_VAL_MO` (import value), `CAL_DUT_MO` (duties collected) |
| `data-input/hs10_classification_final_v3.csv` | LLM-generated classification. Key columns: `hs10_code`, `relevance` (High/Low), `primary_category` |

### Processing Steps

1. Renames and casts columns; parses `time` as monthly datetime.
2. Drops volatile/special HS2 chapters: **27** (energy), **71** (precious metals), **98**, **99** (special categories).
3. Merges in the HS10 classification on `HS10`.
4. Filters to **2023-01-01 onwards**.
5. Computes three metrics for each of the 9 series below:
   - **Dollars ($B)** — monthly import value in billions
   - **Index (2023 = 100)** — annualized monthly value relative to the 2023 total
   - **Tariff Rate (%)** — effective rate as `duties / imports × 100`

### Series Computed

| Series | Definition |
|---|---|
| AI Related | All `relevance == "High"` codes (645 HS10 codes) |
| Non-AI Related | All `relevance == "Low"` codes |
| Compute Hardware | High-relevance, `primary_category == Compute_Hardware` |
| Electrical Power | High-relevance, `primary_category == Electrical_Power` |
| Networking Telecom | High-relevance, `primary_category == Networking_Telecom` |
| Cooling HVAC | High-relevance, `primary_category == Cooling_HVAC` |
| Building Structure | High-relevance, `primary_category == Building_Structure` |
| Fire Safety Security | High-relevance, `primary_category == Fire_Safety_Security` |
| Specialty Materials | High-relevance, `primary_category == Specialty_Materials` |

### Output

`data/ai_trade_series.parquet` — one row per month, 27 columns (3 metrics × 9 series), indexed by `date`.

### How to Run

```bash
python prep_data.py
```

The script will print a row/column count on success. If `data-input/ai_trade_index_series.csv` is present, it also validates the AI Related and Non-AI Related index series against that reference file and prints a PASS/WARNING result.

---

## Bokeh Application

`main.py` is a self-contained Bokeh server application. It loads `data/ai_trade_series.parquet` at startup and renders an interactive chart.

### Running Locally

```bash
bokeh serve --show main.py
```

This opens the app in your browser at `http://localhost:5006/main`.

### Controls

| Control | Description |
|---|---|
| **Display** (Select) | Switch between *Index (2023 = 100)*, *Dollars ($B)*, and *Tariff Rate (%)* |
| **Series** (MultiChoice) | Add or remove any of the 9 series from the chart |
| **Generate CSV Download Link** | Builds a base64-encoded CSV for the currently selected series and mode; click the green link to download |
| **Legend** | Click legend entries on the chart to hide/show individual lines |

### Chart Toolbar

The toolbar below the chart provides: box zoom, reset, pan, x-wheel zoom, and **save as PNG**.

### Tooltips

Hovering over any line shows the series name, formatted date (e.g. `Mar 2025`), and the value in the current display mode.

### Deployment (Heroku)

The `Procfile` starts the app on Heroku:

```
web: bokeh serve --port=$PORT --allow-websocket-origin=<heroku-app-host> --address=0.0.0.0 --use-xheaders main.py
```

---

## Setup

```bash
pip install -r requirements.txt   # bokeh==3.8.0, pandas==2.2.3, pyarrow==18.1.0
python prep_data.py               # build data/ai_trade_series.parquet
bokeh serve --show main.py        # launch app
```
