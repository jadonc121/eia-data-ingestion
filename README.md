# eia-data-ingestion

A Python toolkit for exploring and pulling data from the [EIA Open Data API v2](https://www.eia.gov/opendata/). It was originally built as the energy-data ingestion layer of a causal-inference project, then split out into its own repo so it can be developed further.

> **Status:** work in progress. The core functions work, but several docstrings and edge-case checks are still marked `TODO`.

## What it does

The EIA API organizes data as a tree of *routes* (e.g. `electricity` > `retail-sales`). The module in `src/ingestion/eia_api_ingestion.py` has two public entry points:

- **`view_eia_data`** explores that tree. Pass a list of route IDs (or none, to list the top-level routes). It validates the route lineage, then shows either the available subroutes or, for a terminal route, the dataset's metadata (data columns, facets, frequency, date range, row count). Optionally it also shows facet values.
- **`get_eia_data`** pulls the data. It takes a terminal route, the data columns, and optional facet filters, frequency, date range, sorting and row limits. It validates the request against the route's metadata, batches around the API's 5,000-row page limit, and either returns a pandas DataFrame or writes it to CSV.

[`EIA_CALL_SCHEMATIC.md`](EIA_CALL_SCHEMATIC.md) maps out the internal call tree.

## Setup

Requires Python >= 3.13 and [uv](https://docs.astral.sh/uv/). You also need a free [EIA API key](https://www.eia.gov/opendata/register.php).

```bash
uv sync
```

## Usage

Run from the repo root so the `src` imports resolve (e.g. in a notebook or `uv run python`).

```python
from src.ingestion.eia_api_ingestion import view_eia_data, get_eia_data

API_KEY = "your-eia-api-key"

# 1. Explore: list top-level routes, then narrow down
view_eia_data(API_KEY)
view_eia_data(API_KEY, route_ids=["electricity", "retail-sales"])

# 2. Pull data from a terminal route
df = get_eia_data(
    api_key=API_KEY,
    route_ids=["electricity", "retail-sales"],
    data_columns=["price"],
    freq_filter="monthly",
    start_date_filter="2020-01",
)
```

## Layout

```
src/ingestion/eia_api_ingestion.py   # the pipeline
src/ingestion/common.py              # shared CSV-writing helper
EIA_CALL_SCHEMATIC.md                # call tree of the pipeline
```

## History

This repo's history was extracted from the original `causal-inference` repo with `git filter-repo`, so earlier commits reflect that project's early setup.
