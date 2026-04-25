# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a skill-based travel planning system that generates personalized travel guides and writes them to Notion databases. The system uses a multi-skill orchestration pattern with four specialized skills that work together sequentially.

## Architecture

### Skill Orchestration Pattern

The main workflow follows this sequence:

1. **travel-guide** (orchestrator) - Entry point skill that coordinates the entire workflow
2. **travel-intake** - Collects user preferences, constraints, and travel boundaries
3. **travel-research** - Gathers evidence from multiple data sources
4. **travel-publisher** - Generates Notion pages and performs final validation

Each skill is self-contained in `.claude/skills/<skill-name>/SKILL.md` with optional reference files in `references/` subdirectories.

### Data Sources and Tools

The project integrates three MCP servers and one CLI tool:

- **xiaohongshu-mcp** (HTTP): Real user travel notes and reviews from Xiaohongshu
- **amap-maps** (stdio): Gaode Maps for geocoding, POI search, routing, distance calculation, and weather
- **notion** (stdio): Notion API for database and page operations
- **flyai-cli** (command-line): Fliggy travel data for flights, hotels, attractions, and bookings

**Critical constraint**: Skills must call `flyai-cli` directly via bash commands. Do NOT depend on or reference an external `flyai` skill.

## MCP Configuration

Project-level MCP configuration lives in `.mcp.json` (git-ignored, contains credentials). To set up:

```bash
cp .mcp.example.json .mcp.json
# Edit .mcp.json with your credentials
```

Required credentials:
- `AMAP_MAPS_API_KEY`: Gaode Maps Web Service API key
- `OPENAPI_MCP_HEADERS`: Notion Integration token (format: `{"Authorization": "Bearer YOUR_TOKEN", "Notion-Version": "2022-06-28"}`)
- xiaohongshu-mcp URL: Default `http://localhost:18060/mcp`

### Starting xiaohongshu-mcp

If xiaohongshu-mcp connection fails or requires first-time login:

```bash
./xiaohongshu/xiaohongshu-mcp-darwin-arm64 -headless=false
```

Keep this process running, then retry MCP tool calls.

## flyai-cli Setup

Install globally:

```bash
npm i -g @fly-ai/flyai-cli
```

Verify installation:

```bash
flyai --help
flyai keyword-search --query "Hangzhou 3-day trip"
```

Expected output: JSON with `status: 0` and `message: success`.

### Common flyai-cli Commands

```bash
# Keyword search
flyai keyword-search --query "杭州 3日游"

# Hotel search
flyai search-hotel --dest-name "杭州" --hotel-types "高端酒店" \
  --check-in-date "2026-05-01" --check-out-date "2026-05-03"

# Flight search
flyai search-flight --origin "北京" --destination "杭州" --dep-date "2026-05-01"

# POI/attraction search
flyai search-poi --city-name "杭州" --keyword "西湖 门票"
```

## Skill Development Guidelines

### When to Use Each Skill

- **travel-guide**: User requests travel planning, itinerary generation, or Notion guide creation
- **travel-intake**: Need to collect user preferences before research (missing destination, dates, or constraints)
- **travel-research**: Need to search Xiaohongshu notes, Gaode Maps data, or Fliggy products
- **travel-publisher**: Ready to write structured guide to Notion after research is complete

### Quality Requirements

All generated guides must satisfy the dimensions checklist in `.claude/skills/travel-guide/dimensions.md`:

1. Personal preference modeling
2. Inspiration and attraction discovery
3. Destination deep research (visa, weather, safety)
4. Transportation and accommodation booking
5. Itinerary and pacing planning
6. Budget and financial management
7. Cultural and social integration
8. Supplies and emergency timeline

### Evidence and Attribution Rules

- **Xiaohongshu notes**: Include title and jump link
- **Fliggy products**: Include booking link, mark prices as "参考价格，以实时为准"
- **Gaode Maps**: Cite source for routing, distance, and weather data
- **Notion output**: Return page URL after publishing
- **Images**: Record object, image URL, source link, and usage suggestion; never fabricate sources

### Hard Constraints

- Do NOT arrange daily itinerary until departure and return boundaries are confirmed
- Lock down major transportation first, then arrange local transfers, hotels, and daily activities
- Group by core attractions/activities, then filter outward for hotels, restaurants, and commutes
- Key movements (airport/station ↔ hotel ↔ attraction ↔ restaurant) must use Gaode Maps MCP for method, distance, and estimated time
- Provide indoor backup plans for weather warnings (high temperature, heavy rain, typhoon)
- Notion database uses gallery view; selected attractions, restaurants, and hotels should include images with source attribution

## File Structure

```
.claude/skills/
├── README.md                    # Skill directory overview
├── travel-guide/
│   ├── SKILL.md                 # Main orchestrator
│   └── dimensions.md            # Required quality dimensions
├── travel-intake/
│   └── SKILL.md                 # Preference collection
├── travel-research/
│   ├── SKILL.md                 # Multi-source research
│   └── references/              # Detailed research templates
│       ├── 00-tool-contracts.md
│       ├── 01-transport-boundary.md
│       ├── 02-attractions-activities.md
│       ├── 03-hotels-dining.md
│       ├── 04-routing-weather-risk.md
│       └── 05-evidence-output.md
└── travel-publisher/
    ├── SKILL.md                 # Notion integration
    └── notion-output.md         # Output format spec
```

## Verification Checklist

After setup, verify in order:

```bash
# Validate MCP config JSON
python -m json.tool .mcp.json >/dev/null

# Verify flyai-cli
flyai --help
flyai keyword-search --query "Shanghai cruise"

# Check MCP tools in Claude Code
# Reload project and verify xiaohongshu-mcp, amap-maps, and notion tools are available
```

## What NOT to Commit

- `.mcp.json` (contains credentials)
- Notion tokens or Gaode Maps API keys
- Local login state, caches, or personal travel profiles
- Any files containing sensitive user data
