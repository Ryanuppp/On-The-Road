---
name: google-maps-route-planning
description: >-
  Plan travel routes using Google Maps MCP tools. Use when the user asks to
  plan routes, calculate distances, estimate travel times, find places,
  or generate Google Maps navigation links for a trip.
---

# Google Maps 路线规划

通过 Google Maps MCP 查询实际距离、时间，搜索景点信息，并生成可点击的 Google Maps 导航链接。

## 前置条件

- Google Maps MCP 已配置（`~/.cursor/mcp.json` 中包含谷歌地图相关服务）
- MCP server 名称可能不是英文，需用 `Glob` 扫描 mcps 文件夹确认实际 server identifier

确认 server 名称：

```
1. Glob: mcps 目录下的 SERVER_METADATA.json
2. 读取 serverIdentifier 字段 → 这就是 CallMcpTool 的 server 参数
```

## 可用工具

| 工具 | 用途 | 必填参数 |
|------|------|----------|
| `maps_directions` | 两点间路线（距离、时间、逐步导航） | `origin`, `destination`；可选 `mode` |
| `maps_distance_matrix` | 多起终点距离/时间矩阵 | `origins[]`, `destinations[]`；可选 `mode` |
| `maps_search_places` | 搜索地点（名称、坐标、评分、place_id） | `query`（⚠️ `location` 参数不可用） |
| `maps_place_details` | 地点详情（营业时间、地址等） | `place_id` |
| `maps_geocode` | 地址 → 经纬度 | `address` |
| `maps_reverse_geocode` | 经纬度 → 地址 | `latitude`, `longitude` |
| `maps_elevation` | 海拔查询 | `locations[{latitude, longitude}]` |

`mode` 可选值：`driving`（默认）、`walking`、`bicycling`、`transit`

## 路线规划流程

### Step 1: 搜索景点获取坐标

先用 `maps_search_places` 获取每个景点的精确坐标和 `place_id`：

```
server: <server_identifier>
tool: maps_search_places
args: {"query": "Tokarevsky Lighthouse Vladivostok"}
```

返回值中关注：
- `location.lat` / `location.lng` — 用于后续路线查询和构建链接
- `place_id` — 用于查询详情
- `formatted_address` — 确认是否为目标地点

**重要**：`location` 和 `radius` 参数虽在 schema 中定义但实际不可用，只传 `query`。

### Step 2: 查询逐段路线

用坐标（而非地名）调用 `maps_directions`，避免地名解析错误：

```
server: <server_identifier>
tool: maps_directions
args: {
  "origin": "43.1278,131.9031",
  "destination": "43.0731,131.8431",
  "mode": "driving"
}
```

返回值中提取：
- `routes[0].distance.text` — 距离（如 "11.1 km"）
- `routes[0].duration.text` — 时间（如 "25 mins"）
- `routes[0].steps` — 逐步导航指令

**批量查询技巧**：多段路线可并行调用多个 `maps_directions`，一次性获取全天数据。

### Step 3: 批量距离矩阵（可选）

当需要比较多个起终点组合时，用 `maps_distance_matrix`：

```
server: <server_identifier>
tool: maps_distance_matrix
args: {
  "origins": ["43.1278,131.9031", "43.1175,131.8985"],
  "destinations": ["43.1133,131.8913", "43.1112,131.8816"],
  "mode": "walking"
}
```

**注意**：地名解析可能将多个不同地点解析为同一城市级别坐标（返回 0 距离），务必用坐标查询。

### Step 4: 生成 Google Maps 链接

#### 全程路线链接（多途经点）

```
https://www.google.com/maps/dir/POINT_A/POINT_B/POINT_C/POINT_D
```

支持地名或坐标：

```
https://www.google.com/maps/dir/Novotel+Vladivostok/Tokarevsky+Lighthouse,+Vladivostok/Glass+Beach,+Vladivostok
```

#### 逐段导航链接（A → B）

驾车模式（默认）：
```
https://www.google.com/maps/dir/43.1278,131.9031/43.0731,131.8431
```

步行模式（添加 `!3e2` 参数）：
```
https://www.google.com/maps/dir/43.1278,131.9031/43.1175,131.8985/data=!4m2!4m1!3e2
```

#### 地点直达链接

```
https://www.google.com/maps/search/?api=1&query=43.1278,131.9031
```

## 输出格式

### 交通详情表

每天的路线数据整理为表格：

```markdown
| 路段 | 方式 | 距离 | 时间 | 预估费用 |
| --- | --- | --- | --- | --- |
| A → B | 🚕 打车 | 11.1 km | 25 min | 300–400 ₽ |
| B → C | 🚶 步行 | 1.5 km | 21 min | — |
```

### 导航链接表

每天提供两层链接：

1. **全程路线** — 一键查看当天完整行程
2. **逐段导航** — 每段路独立导航链接

```markdown
> 🗺️ **全程路线**：[查看完整路线](https://www.google.com/maps/dir/...)

**逐段导航链接**：

| # | 路段 | 模式 | 导航 |
| --- | --- | --- | --- |
| ① | A → B | 🚕 驾车 | [导航](https://www.google.com/maps/dir/lat1,lng1/lat2,lng2) |
| ② | B → C | 🚶 步行 | [导航](https://www.google.com/maps/dir/lat1,lng1/lat2,lng2/data=!4m2!4m1!3e2) |
```

### 坐标速查表

文档末尾汇总所有景点坐标，方便随时查询：

```markdown
| 景点 | 坐标 | Google Maps |
| --- | --- | --- |
| XXX | 43.1278, 131.9031 | [打开](https://www.google.com/maps/search/?api=1&query=43.1278,131.9031) |
```

## 费用估算

Google Maps 不返回打车费用。根据当地出租车费率手动计算：

```
预估费用 = 起步价 + 距离(km) × 每公里单价 + 时间(min) × 每分钟单价
```

在文档中注明费率来源和币种。

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| 地名解析到错误地点 | 改用坐标查询，先 `maps_search_places` 获取坐标 |
| `maps_search_places` 的 `location` 参数报错 | 只传 `query`，不传 `location` 和 `radius` |
| 距离矩阵返回 0 | 多个地名解析为同一城市，改用坐标 |
| 路线经过渡轮导致时间异常 | 检查 steps 中是否有 "ferry"，必要时调整起终点坐标 |
| 步行时间比预期长 | Google 考虑了坡度/楼梯，实际可能略快 |
