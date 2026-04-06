# 🛤️ On the Road（在路上）

An AI-powered travel guide generator. Using Cursor AI + MCP toolchain, it automatically collects travel tips from Xiaohongshu (小红书), plans routes via map APIs, generates structured Markdown itineraries, and presents them as a polished single-page application.

AI 驱动的旅行攻略生成器。通过 Cursor AI + MCP 工具链，自动从小红书采集攻略、调用地图 API 规划路线，生成结构化的 Markdown 旅行计划，并以精美的单页应用呈现。

## Preview

Open `index.html` to browse all travel plans. Each guide includes:

- Day-by-day timeline with checkbox progress tracking
- Real-time currency converter (for international destinations)
- Navigation links for every leg (Amap / Google Maps one-click)
- 2–3 restaurant recommendations per meal, with Dianping & map links
- Collapsible attraction details, cost breakdowns, coordinate lookups
- Ad-filtered Xiaohongshu sources — only verified personal experience posts

## Project Structure

```
on-the-road/
├── index.html          # SPA that renders all travel plans
├── plans.json          # Plan registry (title, dates, description)
├── plans/              # Markdown travel guides
│   ├── Vladivostok.md  # Vladivostok 5-day plan (海参崴)
│   └── Suzhou.md       # Suzhou 3-day family plan (苏州)
└── .cursor/skills/     # Cursor AI skill definitions
    ├── travel-guide-creation/        # End-to-end guide generation
    ├── xiaohongshu-search/           # Xiaohongshu search + ad filtering
    └── google-maps-route-planning/   # Google Maps route planning
```

## Workflow

```
Xiaohongshu search (collect attractions / food / hotel tips)
        ↓  filter out ads, keep personal posts only
Map APIs (geocoding + route planning + distance calculation)
        ↓
Generate Markdown guide (strict template format)
        ↓
Register in plans.json → index.html auto-renders
        ↓
Chrome DevTools screenshot verification
```

## MCP Toolchain

This project uses the following MCP (Model Context Protocol) services:

| MCP Service | Purpose | Key Tools |
| --- | --- | --- |
| **Amap (高德地图)** | Domestic route planning, geocoding, POI search | `maps_geo` `maps_direction_driving` `maps_direction_walking` `maps_direction_transit_integrated` |
| **Google Maps** | International route planning, place search, distance matrix | `maps_directions` `maps_search_places` `maps_distance_matrix` `maps_geocode` |
| **RedNote search (小红书)** | Search Xiaohongshu notes, extract guides and comments | `search_notes` `get_note_content` `get_note_comments` |
| **Chrome DevTools** | Browser automation for page verification and fallback search | `new_page` `take_screenshot` `take_snapshot` `evaluate_script` |

## Cursor Skills

The project includes three custom Cursor skills (`.cursor/skills/`) that define the AI's complete workflow:

- **travel-guide-creation** — End-to-end guide generation: from data collection to formatted output
- **xiaohongshu-search** — Xiaohongshu search strategy with image content extraction, ad filtering rules, and Chrome DevTools fallback
- **google-maps-route-planning** — Google Maps route queries, cost estimation, and navigation link generation

## Run Locally

```bash
cd on-the-road
python3 -m http.server 8765
# Open http://localhost:8765
```

## License

MIT
