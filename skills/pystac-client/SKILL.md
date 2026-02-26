---
name: pystac-client
description: Search satellite imagery catalogs using pystac-client. Use for STAC catalog searches, finding Sentinel/Landsat imagery, querying CDSE or other STAC APIs.
argument-hint: "[search-description or collection-name]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# PySTAC Client - STAC Catalog Search Skill

You help users search satellite imagery catalogs using the `pystac-client` library and its `stac-client` CLI.

## Prerequisites

Ensure `pystac-client` is installed in the active Python environment:

```bash
pip install pystac-client
```

Verify with:
```bash
stac-client --help
```

## Well-Known STAC Endpoints

| Name | URL | Auth |
|------|-----|------|
| **CDSE** (Copernicus Data Space) | `https://stac.dataspace.copernicus.eu/v1/` | No (search) |
| **Earth Search** (AWS) | `https://earth-search.aws.element84.com/v1` | No |
| **Planetary Computer** | `https://planetarycomputer.microsoft.com/api/stac/v1` | No (search) |

If the user does not specify an endpoint, ask which one to use.

## Common Collections

| Collection ID | Description | Available On |
|---------------|-------------|--------------|
| `sentinel-2-l2a` | Sentinel-2 Surface Reflectance | CDSE, Earth Search, PC |
| `sentinel-2-l1c` | Sentinel-2 Top of Atmosphere | CDSE, Earth Search |
| `sentinel-1-grd` | Sentinel-1 Ground Range Detected | CDSE, Earth Search, PC |
| `sentinel-1-slc` | Sentinel-1 Single Look Complex | CDSE |
| `landsat-c2-l2` | Landsat Collection 2 Level-2 | Earth Search, PC |
| `cop-dem-glo-30-dged-cog` | Copernicus DEM 30m | CDSE |

Collection IDs vary across endpoints. Use `stac-client collections <URL>` to discover what is available.

## How to Respond

Based on the user's request (`$ARGUMENTS`), choose the appropriate approach:

### 1. List Available Collections

```bash
stac-client collections <URL>
```

Or with free-text search:
```bash
stac-client collections <URL> -q "sentinel-2"
```

### 2. Quick Result Count

Use `--max-items` with `jq` to count results. Not all STAC APIs support `--matched`, so prefer this approach:

```bash
stac-client search <URL> \
  -c <collection> \
  --bbox <minlon> <minlat> <maxlon> <maxlat> \
  --datetime <start>/<end> \
  --max-items 500 | jq '.features | length'
```

### 3. Search and Display Results

```bash
stac-client search <URL> \
  -c <collection> \
  --bbox <minlon> <minlat> <maxlon> <maxlat> \
  --datetime <start>/<end> \
  --query "eo:cloud_cover<20" \
  --max-items 10 \
  | jq '.features[] | {id: .id, date: .properties.datetime, cloud: .properties["eo:cloud_cover"], platform: .properties.platform}'
```

### 4. Save Results to File

```bash
stac-client search <URL> \
  -c <collection> \
  --bbox <minlon> <minlat> <maxlon> <maxlat> \
  --datetime <start>/<end> \
  --max-items 50 \
  --save results.geojson
```

### 5. Python Script (for complex workflows)

When the CLI is not sufficient, write a Python script:

```python
from pystac_client import Client

client = Client.open("<URL>")

search = client.search(
    collections=["<collection>"],
    bbox=[minlon, minlat, maxlon, maxlat],
    datetime="<start>/<end>",
    query={"eo:cloud_cover": {"lt": 20}},
    max_items=10,
    sortby="-properties.datetime",
)

for item in search.items():
    props = item.properties
    print(f"{item.id} | {props['datetime']} | cloud: {props.get('eo:cloud_cover')}%")

# Save as GeoJSON
search.item_collection().save_object("results.geojson")
```

## Python API Reference

### Client.open()

```python
client = Client.open(
    url,                        # STAC API root URL
    headers=None,               # dict - HTTP headers for auth
    timeout=None,               # float or (connect, read) tuple in seconds
    modifier=None,              # callable - modify returned STAC objects in-place
    request_modifier=None,      # callable - modify requests before sending
    stac_io=None,               # StacApiIO - custom IO with retry/timeout
)
```

### client.search()

```python
search = client.search(
    collections=None,           # str|list - Collection ID(s)
    bbox=None,                  # list - [minlon, minlat, maxlon, maxlat]
    intersects=None,            # dict|Shapely - GeoJSON geometry (exclusive with bbox)
    datetime=None,              # str - "2024-01-01/2024-06-01" or "2024-01-01/.."
    ids=None,                   # str|list - Specific item IDs (overrides other filters)
    query=None,                 # dict - {"eo:cloud_cover": {"lt": 10}}
    filter=None,                # dict|str - CQL2 filter expression
    filter_lang=None,           # str - "cql2-json" or "cql2-text"
    sortby=None,                # str|list - "-properties.datetime" (desc), "+id" (asc)
    fields=None,                # str|list|dict - include/exclude fields
    max_items=None,             # int - max total items across all pages
    limit=None,                 # int - items per page (default 100)
    method="POST",              # str - "POST" or "GET"
)
```

**Query operators:** `eq`, `neq`, `lt`, `lte`, `gt`, `gte`

**Datetime formats:**
- Single: `"2024-01-01"`, `"2024-01"`, `"2024"`
- Range: `"2024-01-01/2024-06-01"`
- Open-ended: `"2024-01-01/.."`, `"../2024-06-01"`

### Result Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `search.items()` | Iterator[Item] | Lazy paginated iterator of pystac Items |
| `search.items_as_dicts()` | Iterator[dict] | Raw dicts (faster, no parsing) |
| `search.item_collection()` | ItemCollection | All results loaded into memory |
| `search.item_collection_as_dict()` | dict | GeoJSON FeatureCollection |
| `search.pages()` | Iterator[ItemCollection] | Page-by-page iteration |
| `search.matched()` | int or None | Server-side match count (not supported by all APIs) |

### Collection Search

```python
results = client.collection_search(q="sentinel", bbox=[...], datetime="...")
for c in results.collections():
    print(c.id, c.title)
```

### CQL2 Filtering

```python
# CQL2-JSON (for POST, default)
search = client.search(
    collections=["sentinel-2-l2a"],
    filter={
        "op": "and",
        "args": [
            {"op": "<=", "args": [{"property": "eo:cloud_cover"}, 10]},
            {"op": ">=", "args": [{"property": "datetime"}, "2024-01-01"]},
        ]
    },
)

# CQL2-TEXT (for GET)
search = client.search(
    method="GET",
    collections=["sentinel-2-l2a"],
    filter="eo:cloud_cover<=10",
)
```

### Authentication

```python
# Bearer token
client = Client.open(url, headers={"Authorization": "Bearer <token>"})

# API key
client = Client.open(url, headers={"X-Api-Key": "<key>"})

# Planetary Computer (auto-sign asset URLs)
import planetary_computer
client = Client.open(url, modifier=planetary_computer.sign_inplace)
```

### Retry and Timeout

```python
from pystac_client.stac_api_io import StacApiIO
from urllib3.util.retry import Retry

retry = Retry(total=5, backoff_factor=1, status_forcelist=[502, 503, 504])
stac_io = StacApiIO(max_retries=retry, timeout=30)
client = Client.open(url, stac_io=stac_io)
```

### Convert to GeoDataFrame

```python
import geopandas as gpd
items = search.item_collection()
gdf = gpd.GeoDataFrame.from_features(items.to_dict())
```

## CLI Reference

### stac-client search

```
stac-client search <URL> [OPTIONS]
  -c, --collections    Collection ID(s)
  --bbox               minlon minlat maxlon maxlat
  --intersects         GeoJSON geometry (file path or string)
  --datetime           YYYY-MM-DD or YYYY-MM-DD/YYYY-MM-DD
  --ids                Specific item IDs
  --query              Property filter ("eo:cloud_cover<10")
  --filter             CQL2 filter expression
  --filter-lang        cql2-json or cql2-text
  --sortby             Sort fields (+field ascending, -field descending)
  --fields             Include/exclude fields (+field or -field)
  --max-items          Max items to return
  --limit              Page size
  --matched            Print match count only (not supported by all STAC APIs)
  --save               Save output to file
  --headers            HTTP headers as KEY=VALUE (repeatable)
  --method             GET or POST (default: POST)
```

### stac-client collections

```
stac-client collections <URL> [OPTIONS]
  -q, --q              Free-text search
  --bbox               Bounding box filter
  --datetime           Temporal filter
  --save               Save to file
```

## CLI Tips

- `--fields` uses space-separated field names for includes: `--fields properties.datetime properties.eo:cloud_cover`
- Exclude fields are not supported as CLI args (use the Python API `fields={"exclude": [...]}` instead)
- Pipe to `jq` for formatting: `| jq '.features[] | {id, date: .properties.datetime}'`
- Count results with: `| jq '.features | length'`

## Best Practices

- Always use `--max-items` to avoid accidentally fetching thousands of results
- Use `| jq '.features | length'` to count results instead of `--matched` (which many STAC APIs do not support)
- Cloud cover filtering (`eo:cloud_cover`) is essential for optical imagery (Sentinel-2, Landsat)
- Sort by `-properties.datetime` to get most recent imagery first
- Use `intersects` with a GeoJSON geometry for precise AOI filtering instead of bbox
- Collection IDs differ between STAC endpoints; always verify with `stac-client collections`
- STAC search endpoints are typically public; authentication is usually only required for data download
