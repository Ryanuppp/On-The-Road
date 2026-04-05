# 🛤️ On the Road（在路上）

一个 AI 驱动的旅行攻略生成器。通过 Cursor AI + MCP 工具链，自动从小红书采集攻略、调用地图 API 规划路线，生成结构化的 Markdown 旅行计划，并以精美的单页应用呈现。

## 预览

打开 `index.html` 即可浏览所有旅行计划。每份攻略包含：

- 逐日行程时间线，支持 checkbox 勾选追踪进度
- 实时汇率换算器（境外目的地）
- 每段路线的导航链接（高德/Google Maps 一键跳转）
- 每餐 2–3 家餐厅推荐卡片，附大众点评和地图链接
- 景点详情、交通费用总览、坐标速查表

## 项目结构

```
on-the-road/
├── index.html          # 单页应用，渲染所有旅行计划
├── plans.json          # 计划注册表（标题、日期、描述）
├── plans/              # Markdown 格式的旅行攻略
│   ├── Vladivostok.md  # 海参崴 5 日计划
│   └── Suzhou.md       # 苏州 3 日亲子计划
└── .cursor/skills/     # Cursor AI 技能定义
    ├── travel-guide-creation/   # 攻略生成全流程
    ├── xiaohongshu-search/      # 小红书搜索与内容提取
    └── google-maps-route-planning/  # Google Maps 路线规划
```

## 工作流程

```
小红书搜索（采集攻略/美食/住宿推荐）
        ↓
地图 API（地理编码 + 路线规划 + 距离计算）
        ↓
生成 Markdown 攻略（严格模板格式）
        ↓
注册到 plans.json → index.html 自动展示
        ↓
Chrome DevTools 截图验证渲染效果
```

## MCP 工具链

本项目使用以下 MCP（Model Context Protocol）服务，均可从 [MCP Market](https://mcpmarket.cn/) 获取：

| MCP 服务 | 用途 | 核心工具 |
| --- | --- | --- |
| **高德地图 (Amap)** | 国内路线规划、地理编码、POI 搜索 | `maps_geo` `maps_direction_driving` `maps_direction_walking` `maps_direction_transit_integrated` `maps_text_search` |
| **谷歌地图 (Google Maps)** | 境外路线规划、景点搜索、距离矩阵 | `maps_directions` `maps_search_places` `maps_distance_matrix` `maps_geocode` |
| **RedNote search (小红书)** | 搜索小红书笔记、获取攻略内容和评论 | `search_notes` `get_note_content` `get_note_comments` |
| **Chrome DevTools** | 浏览器自动化，用于页面验证和小红书备用搜索 | `new_page` `take_screenshot` `take_snapshot` `evaluate_script` |

## Cursor Skills

项目包含三个自定义 Cursor 技能（`.cursor/skills/`），定义了 AI 的完整工作流：

- **travel-guide-creation** — 端到端攻略生成流程，从信息采集到格式化输出
- **xiaohongshu-search** — 小红书搜索策略，含图片内容读取和 Chrome DevTools 备用方案
- **google-maps-route-planning** — Google Maps 路线查询、费用估算和导航链接生成

## 本地运行

```bash
cd on-the-road
python3 -m http.server 8765
# 浏览器打开 http://localhost:8765
```

## License

MIT
