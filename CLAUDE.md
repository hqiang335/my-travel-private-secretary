# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **旅游私人秘书 (Travel Private Secretary)** — an AI-driven workflow that takes a user's travel request and produces a fully formatted Notion itinerary page. It is not a software application; it is a collection of skill definitions, reference docs, and user preference profiles that guide Claude through a 5-phase execution pipeline.

## Triggering the Workflow

Invoke the skill via `/travel-secretary` or when the user asks to plan a trip. The skill definition lives at [.claude/skills/travel-secretary/SKILL.md](.claude/skills/travel-secretary/SKILL.md).

## 5-Phase Execution Pipeline

### Phase 0 — Pre-flight checks
- Verify `xiaohongshu-mcp` is running (`ps aux | grep xiaohongshu-mcp`). It must be started manually; first run requires QR-code login.
- Ensure profile directory exists: `.claude/travel-profiles/`

### Phase 1 — Smart questionnaire
- Read existing profiles from `.claude/travel-profiles/*.json`; offer to reuse.
- Extract known info from the user's message before asking questions.
- Ask in batches of ≤ 4 questions using `AskUserQuestion`. **Never include `_` fields in the `filters` object** (causes MCP error -32602).
- Save profile to `.claude/travel-profiles/{destination}_{YYYYMMDD}.json`.

### Phase 2 — Parallel data collection
Run all three sources in parallel:

| Source | Tool | Key params |
|--------|------|-----------|
| 小红书 | `xiaohongshu-mcp` search tools | 4 searches: 攻略/住宿/美食/避坑 |
| 高德地图 | `amap-maps` MCP tools | weather, geo, distance |
| 飞猪 | `flyai` CLI (Bash) | see exact param names below |

**flyai CLI exact parameter names** (wrong names silently fail):
```bash
flyai search-hotel --dest-name "城市" --hotel-types "类型" --check-in-date "日期" --check-out-date "日期"
flyai search-flight --origin "出发城市" --destination "目的地" --dep-date "日期"
flyai search-poi --city-name "城市" --keyword "景点名 门票"
```

### Phase 3 — Itinerary planning
- Cluster attractions by geographic proximity (use amap distance data).
- Match daily count to pacing: 特种兵 4-5/day, 适中 2-3/day, 度假 1/day.
- Structure each day: 上午 → 午餐 → 下午 → 晚餐 → 夜间 → 住宿.

### Phase 4 — Notion page generation
Use **Python `urllib`** to call Notion REST API directly (do not use Notion MCP — requires restart).

Read token from `~/.claude.json`:
```python
config["mcpServers"]["notion"]["env"]["OPENAPI_MCP_HEADERS"]  # parse Bearer token
```

**Critical limit**: Notion API accepts ≤ 100 blocks per request. Split into batches:
- Batch 1 (≤ 95 blocks): create page via `POST /v1/pages`
- Batch 2+: append via `PATCH /v1/blocks/{page_id}/children`

Target database: `🗺️ 旅游攻略库` — search first, create if missing.

Page structure (in order): cover image → overview table → weather/booking callout → transport → Day 1/2/3 → hotels → ticket table → budget table → 避坑 callout → todo checklist → 小红书 sources.

### Phase 5 — Wrap-up
- Append `last_generated`, `notion_page_id`, `notion_page_url`, `notion_db_id` to the profile JSON.
- Update `memory/notion-config.md` with the database ID.
- Report the Notion page URL and key highlights to the user.

## Key Files

| Path | Purpose |
|------|---------|
| `.claude/skills/travel-secretary/SKILL.md` | Master execution handbook |
| `.claude/skills/travel-secretary/references/` | Per-phase detailed reference docs |
| `.claude/travel-profiles/` | Saved user preference profiles (JSON) |
| `flyai/SKILL.md` | flyai CLI usage and display rules |
| `flyai/references/` | Per-command parameter schemas |
| `偏好初始化问卷.md` | Initial preference questionnaire template |
| `旅游规划全维度深度考量表.xlsx` | Comprehensive planning checklist |

## Notion Block Helpers (reference)

Reuse these patterns when building blocks in Python:
```python
def txt(content, bold=False, link=None): ...
def h1/h2/h3(content): ...
def para(*parts): ...
def bullet(*parts): ...
def callout(content, emoji="📍"): ...
def todo(content, checked=False): ...
def table_row(cells): ...   # table_width must match len(cells)
def image_block(url): ...   # embed 小红书 URLs directly, no download needed
```

## Common Pitfalls

- `AskUserQuestion` filters must not contain `_` fields → MCP error -32602
- flyai param names: `--dest-name` not `--city`; `--origin` not `--from`; `--dep-date` not `--date`
- Notion blocks per request ≤ 100 (use ≤ 95 for safety)
- 小红书 image URLs are time-limited; embed directly into Notion without downloading
- flyai booking links are time-limited; label prices as "参考价格，以实时为准"
