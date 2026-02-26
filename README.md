# pystac-skill - Claude Code Plugin

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin for searching satellite imagery catalogs using [pystac-client](https://github.com/stac-utils/pystac-client).

Ask Claude to find satellite imagery by describing what you need in natural language.

## Install as Plugin

Install directly from GitHub inside Claude Code:

```
/plugin install pystac-skill --source github:Guyon-Duifhuizen/pystac-skill
```

Or add via a marketplace that includes this plugin (see [Distributing via marketplace](#distributing-via-marketplace)).

### Manual Install

Copy the skill into your Claude Code skills directory:

```bash
# Personal (available in all projects)
mkdir -p ~/.claude/skills/pystac-client
cp skills/pystac-client/SKILL.md ~/.claude/skills/pystac-client/

# Or project-level (available in one project)
mkdir -p .claude/skills/pystac-client
cp skills/pystac-client/SKILL.md .claude/skills/pystac-client/
```

### Python Dependency

Ensure `pystac-client` is installed in your Python environment:

```bash
pip install pystac-client
```

## Usage

Invoke with `/pystac-client` followed by your request:

```
/pystac-client list collections on CDSE
/pystac-client find Sentinel-2 images over Amsterdam from last week with low cloud cover
/pystac-client search Landsat imagery over bbox 4.0 52.0 5.0 52.5 for June 2024
/pystac-client save Sentinel-1 GRD results for the Netherlands to a geojson file
```

Claude will use the `stac-client` CLI or write Python scripts depending on the complexity of your request.

## Supported STAC Endpoints

| Name | URL |
|------|-----|
| CDSE (Copernicus Data Space) | `https://stac.dataspace.copernicus.eu/v1/` |
| Earth Search (AWS) | `https://earth-search.aws.element84.com/v1` |
| Planetary Computer | `https://planetarycomputer.microsoft.com/api/stac/v1` |

## What's in the Skill

- CLI reference for `stac-client search` and `stac-client collections`
- Python API reference for `Client`, `ItemSearch`, `CollectionSearch`
- CQL2 filtering, authentication patterns, retry/timeout configuration
- Common collection IDs across endpoints
- Practical tips (cloud cover filtering, `jq` piping, result counting)

## Distributing via Marketplace

To include this plugin in your own marketplace, add it to your `marketplace.json`:

```json
{
  "plugins": [
    {
      "name": "pystac-skill",
      "source": {
        "source": "github",
        "repo": "Guyon-Duifhuizen/pystac-skill"
      },
      "description": "Claude Code skill for searching satellite imagery catalogs via pystac-client"
    }
  ]
}
```

## Project Structure

```
pystac-skill/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/
│   └── pystac-client/
│       └── SKILL.md             # The skill definition
├── pyproject.toml
├── uv.lock
└── README.md
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- Python 3.10+
- `pystac-client` (`pip install pystac-client`)
- `jq` (recommended, for CLI output formatting)

## License

MIT
