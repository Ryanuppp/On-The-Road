---
name: xiaohongshu-search
description: >-
  Search and extract content from Xiaohongshu (小红书). Primary method uses
  RedNote search MCP; fallback uses Chrome DevTools MCP. Use when the user asks
  to search 小红书, browse Xiaohongshu notes, or gather travel guides, reviews,
  or recommendations from 小红书.
---

# 小红书搜索与内容提取

## 方法一：RedNote search MCP（推荐）

通过 RedNote search MCP 直接调用小红书 API 搜索和获取笔记内容，无需操控浏览器，速度快且稳定。

### 前置条件

- `rednote-mcp` 已安装（项目位于 `~/Toys/rednote-mcp`）
- 已运行 `rednote-mcp init` 完成登录，cookie 保存在 `~/.mcp/rednote/cookies.json`
- `~/.cursor/mcp.json` 中已配置：

```json
"RedNote search": {
    "command": "node",
    "args": [
        "/Users/bytedance/Toys/rednote-mcp/dist/cli.js",
        "--stdio"
    ]
}
```

### 常见问题排查

如果 MCP 状态显示 errored：

1. **Playwright 未安装**：在 `~/Toys/rednote-mcp` 下运行 `npx playwright install`
2. **Cookie 过期**：运行 `rednote-mcp init` 重新扫码登录
3. **PATH 问题**：确保 mcp.json 中使用 `node` + 绝对路径（`/Users/bytedance/Toys/rednote-mcp/dist/cli.js`），而非 `rednote-mcp` 命令名
4. 修改配置后需要 `Cmd+Shift+P` → `Reload Window` 重新加载 Cursor

### 可用工具

#### 1. search_notes — 搜索笔记

根据关键词搜索小红书笔记，返回标题、作者、内容摘要、点赞数、评论数和链接。

```
server: user-RedNote search
tool: search_notes
args: {"keywords": "搜索关键词", "limit": 10}
```

- `keywords`（必填）：搜索关键词，如 "五月海参崴攻略"
- `limit`（可选）：返回结果数量，默认不限

返回格式：每条结果包含标题、作者、内容全文、点赞数、评论数、链接，以 `---` 分隔。

#### 2. get_note_content — 获取笔记详情

根据笔记 URL 获取完整内容。

```
server: user-RedNote search
tool: get_note_content
args: {"url": "https://www.xiaohongshu.com/explore/..."}
```

- `url`（必填）：笔记链接，可从 search_notes 结果中获取

#### 3. get_note_comments — 获取笔记评论

获取指定笔记的评论内容。

```
server: user-RedNote search
tool: get_note_comments
args: {"url": "https://www.xiaohongshu.com/explore/..."}
```

#### 4. login — 登录

在 MCP 内触发登录流程（通常不需要，建议用 CLI `rednote-mcp init` 代替）。

```
server: user-RedNote search
tool: login
args: {}
```

### 图片内容读取（重要）

小红书笔记的有效信息经常只在图片中，文字正文可能只有简短的标题或摘要。**当笔记文字内容很短但图片数量较多时，必须读取图片来提取完整信息。**

#### 判断标准

- `get_note_content` 返回的 `content` 很短（< 100 字），但 `imgs` 数组包含多张图片 → 核心内容在图片中
- 笔记类型为"攻略"、"教程"、"清单"、"对比"等信息密集型内容 → 大概率图文笔记，信息在图里

#### 读取流程

1. **从 `get_note_content` 的返回中获取 `imgs` 数组**（webp 格式 URL 列表）

2. **下载图片到临时目录**：
```bash
mkdir -p /tmp/xhs_images
curl -s -o /tmp/xhs_images/img1.webp "<图片URL>"
```

3. **转换格式**（Read 工具不支持 webp 解码，需转为 png）：
```bash
sips -s format png /tmp/xhs_images/img1.webp --out /tmp/xhs_images/img1.png
```

4. **用 Read 工具读取 png 图片**，即可识别图片中的文字内容

5. **清理临时文件**：
```bash
rm -rf /tmp/xhs_images
```

#### 注意事项

- 图片 URL 包含临时 token，获取后应尽快下载，过期后需重新调用 `get_note_content`
- 图片数量较多时（>10 张），可先下载前几张了解内容结构，再决定是否需要读取全部
- 每次批量处理时建议并行下载和转换以提高效率：
```bash
for i in $(seq 1 $N); do
  curl -s -o /tmp/xhs_images/img${i}.webp "$URL" &
done
wait
for f in /tmp/xhs_images/*.webp; do
  sips -s format png "$f" --out "${f%.webp}.png"
done
```
- `sips` 是 macOS 自带工具，无需额外安装

### 搜索技巧

- 多次搜索不同关键词可以获得更全面的结果，例如先搜 "五月海参崴攻略"，再搜 "海参崴 美食推荐 天气 穿衣"
- `search_notes` 返回的内容已包含笔记全文摘要，大多数情况下无需再调用 `get_note_content`
- 如需查看评论中的补充信息（如更新的价格、避雷提醒），可用 `get_note_comments`
- **当笔记正文内容简短时，务必通过图片读取流程获取完整信息**

---

## 方法二：Chrome DevTools MCP（备用）

当 RedNote search MCP 不可用时，可通过 Chrome DevTools MCP 操控浏览器访问小红书网页版。

### 前置条件

- Chrome DevTools MCP 已配置（`~/.cursor/mcp.json` 中包含 `chrome-devtools`）
- Google Chrome 浏览器已安装

### 搜索流程

#### Step 1: 打开搜索页面

使用 `new_page` 打开小红书搜索 URL，关键词需 URL 编码：

```
URL 格式: https://www.xiaohongshu.com/search_result?keyword={关键词}&source=web_search_result_notes
```

MCP 调用：
- server: `user-chrome-devtools`
- tool: `new_page`
- args: `{"url": "https://www.xiaohongshu.com/search_result?keyword=<编码后的关键词>&source=web_search_result_notes"}`

#### Step 2: 处理登录弹窗

小红书搜索页通常弹出登录窗口遮挡内容。用 `evaluate_script` 移除弹窗：

```javascript
() => {
  const overlay = document.querySelector('.login-mask, .overlay, [class*="login"], [class*="mask"]');
  if (overlay) { overlay.remove(); }
  const allModals = document.querySelectorAll('[class*="modal"], [class*="dialog"], [class*="popup"]');
  allModals.forEach(m => m.remove());
  document.body.style.overflow = 'auto';
}
```

移除弹窗后，等待 2-3 秒让页面加载，然后 **reload 页面**以触发完整内容渲染：

- tool: `navigate_page`
- args: `{"type": "reload"}`

再等待 2-3 秒（用 `evaluate_script` 执行 `setTimeout`）。

#### Step 3: 截图确认页面状态

用 `take_screenshot` 确认搜索结果是否正常显示。

#### Step 4: 提取搜索结果

用 `take_snapshot` 获取页面 a11y 树。搜索结果的关键结构：

| 元素类型 | 内容 |
|----------|------|
| `link` (带 `/search_result/` URL) | 笔记入口，uid 用于后续点击 |
| 紧跟的 `StaticText` | 笔记标题 |
| 子 `link` (带 `/user/profile/` URL) | 作者名 + 发布日期 |
| 数字 `StaticText` | 点赞数 |

从 snapshot 中提取这些信息，整理为结构化列表呈现给用户。

#### Step 5: 查看笔记详情（可选）

如需查看某篇笔记详情：

1. 用 `click` 点击笔记链接的 uid
2. 等待页面加载（2-3 秒）
3. 用 `take_snapshot` 提取笔记正文内容
4. 用 `take_screenshot` 截图展示

### 等待页面加载的方法

Chrome DevTools MCP 的 `wait_for` 只支持等待文本出现，不支持纯延时。使用 `evaluate_script` 实现延时：

```javascript
() => { return new Promise(resolve => setTimeout(() => resolve('done'), 3000)); }
```

### 注意事项

- 小红书可能检测自动化访问，如遇到验证码需用户手动处理
- 如果 Chrome 已登录小红书账号，弹窗移除后 reload 会自动使用已有登录状态
- 搜索结果默认按"综合"排序，页面上有筛选标签可通过 `click` 切换
