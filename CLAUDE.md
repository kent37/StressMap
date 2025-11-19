# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is LTS-OSM (Level of Traffic Stress - OpenStreetMap), a Python tool that calculates cycling Level of Traffic Stress for all road and path segments in a region using OpenStreetMap data. The code is adapted from Bike Ottawa's LTS code with modifications to include intersection LTS calculations.

## Common Commands

### Setup
```bash
pip install -r requirements.txt
```

### Processing and Visualization
```bash
# Process OSM data for a city and calculate LTS
python3 main.py process -city Northampton

# Rebuild underlying data from scratch
python3 main.py process -city Northampton --rebuild

# Process and automatically plot
python3 main.py process -city Northampton --plot

# Process multiple cities and combine them
python3 main.py process -cities Boston,Cambridge --combine --plot

# Plot existing processed data
python3 main.py plot -city Northampton

# Plot to GeoJSON format
python3 main.py plot -city Northampton --format=json

# Combine multiple cities into a regional map
python3 main.py combine -cities Boston,Cambridge,Somerville

# Serve the visualization on local web server
python3 web.py
# Then browse to http://localhost:8000
```

### Adding New Cities

Cities are defined in `constants.py`. To add a new city:
1. Find the region on openstreetmap.org
2. Identify the OSM relation key/value (usually wikipedia)
3. Add to the `CITIES` dictionary in `constants.py`

Example:
```python
"CityName": {
    "key": "wikipedia",
    "value": "en:CityName, Massachusetts"
}
```

## Architecture

### Data Processing Pipeline

The codebase uses a multi-stage pipeline where each stage checks for existing output files before processing. To re-run from a specific stage, delete that stage's output file and all subsequent files. Files are numbered in order of generation:

1. **Query Building** (`build_query.py`, `LTS_OSM.build_query`)
   - Creates Overpass API query files in `query/` directory
   - Queries filter for highways, excluding dirt/ground surfaces, parking aisles, and sidewalks (unless explicitly bike-friendly)

2. **OSM Download** (`LTS_OSM.download_osm`)
   - Downloads raw OSM data via Overpass API
   - Output: `data/{region}_1.json`

3. **Tag Extraction** (`LTS_OSM.extract_tags`)
   - Extracts all unique OSM tags from downloaded data
   - Adds tags to osmnx settings for subsequent processing
   - Output: `data/{region}_2_way_tags.csv`

4. **Graph Download** (`LTS_OSM.download_data`)
   - Uses osmnx to download street network with custom filters
   - Converts to GeoDataFrames (nodes and edges)
   - Output: `data/{region}_3.graphml`

5. **LTS Edge Calculation** (`LTS_OSM.lts_edges`)
   - Core LTS calculation for all road segments
   - Uses configuration files in `config/` directory
   - Output: `data/{region}_4_all_lts.csv`

6. **LTS Node Calculation** (`LTS_OSM.lts_nodes`)
   - Calculates intersection LTS based on traffic controls
   - Output: `data/{region}_6_gdf_nodes.csv`

7. **Plotting** (`LTS_plot.py`)
   - Generates visualizations (HTML or GeoJSON)
   - Output: `plots/` directory

### LTS Calculation Logic (`lts_functions.py`)

The LTS calculation is highly configurable via YAML files in `config/`:

- **`tables.yml`**: Implements LTS tables from the academic specification (LTS-Tables-v2.2.pdf)
- **`rating_dict.yml`**: Rules for inferring street characteristics from OSM tags (speed, parking, ADT, etc.)
- **`lane_parse.yml`**: Parses OSM cycleway tags into directional bike lane information

Key calculation steps:
1. Parse directional bike lane features from OSM tags (converts `:both` suffixes to `:left/:right`)
2. Infer missing data (speed limits, lane counts, parking presence, ADT estimates)
3. Calculate segment widths and classify as narrow/wide oneways
4. Evaluate LTS tables based on street characteristics
5. Use minimum LTS across all calculation methods (mixed traffic, bike lanes, separated facilities)
6. Final segment LTS is the maximum of forward/reverse direction LTS

### Configuration System

The rule application system in `lts_functions.apply_rules`:
- Evaluates conditions in order from configuration files
- Uses pandas `.eval()` for dynamic condition evaluation
- Once a rule matches, subsequent rules don't override
- Supports directional (`:left/:right`) and symmetric tagging
- Handles namespace expansion for `[both]`, `[left]`, `[right]` placeholders

### Node (Intersection) LTS

Node LTS logic (`LTS_OSM.lts_nodes`):
- Default: maximum LTS of all intersecting edges
- LTS 3-4 intersections with traffic signals → reduced to LTS 2
- LTS 1-2 intersections with traffic signals or stop signs → reduced to LTS 1

### Data Combination

`LTS_OSM.combine_data` can merge multiple processed cities into a regional dataset for plotting larger areas.

### Web Visualization

`web.py` serves a simple HTTP server that:
- Serves HTML/GeoJSON visualizations at root path
- Defaults to `map/local_data.html` on port 8000
- Allows custom plot paths and ports via CLI arguments

### Isochrone Analysis (Experimental)

`isochrone.py` contains work-in-progress code to generate isochrone maps showing reachable areas at different LTS thresholds. Requires both the graph object and LTS dataframe.

## Important Notes

- The codebase uses a global `OVERWRITE` flag to control whether to regenerate files. Set via `--rebuild` flag or by deleting intermediate files.
- osmnx caching is disabled (`ox.settings.use_cache = False`) to prevent stale data issues with un-separated cycleways.
- All width measurements are converted to decimal feet for consistency.
- LTS scores: 1 (best) to 4 (worst), with 0 indicating no biking permitted.
- Default assumptions err on the high end (higher stress) when data is missing.
- The pipeline is designed for Massachusetts cities but can be adapted for other regions by modifying the `graph_from_place` call in `download_data`.
