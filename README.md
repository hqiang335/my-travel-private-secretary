# 个性化旅游攻略 Skill 项目

这个仓库用于开发一套可复用的旅行规划 Skill：根据用户的个性化需求生成旅游攻略，并写入 Notion Database。

本项目的工具边界是：

- 项目级 MCP 只负责连接小红书、高德地图、Notion。
- 飞猪数据只通过 `flyai-cli` 命令行直接调用。
- Skill 本身不依赖、不引用外部的 `flyai` skill。

## 环境要求

- Node.js 18 或更高版本
- npm
- 可用的 Notion Integration Token
- 可用的高德地图 Web 服务 API Key
- 本地可访问的小红书 MCP 服务

## 安装 `flyai-cli`

全局安装：

```bash
npm i -g @fly-ai/flyai-cli
```

确认命令可用：

```bash
flyai --help
```

执行一次真实查询验证连通性：

```bash
flyai keyword-search --query "Hangzhou 3-day trip"
```

期望结果是 stdout 返回单行 JSON，且包含类似字段：

```json
{
  "status": 0,
  "message": "success",
  "data": {
    "itemList": []
  }
}
```

## 配置项目级 MCP

本项目使用项目级 `.mcp.json`，不要把 MCP 配置放到个人全局配置里。`.mcp.json` 通常包含本地密钥，因此已经被 `.gitignore` 忽略。

从模板复制：

```bash
cp .mcp.example.json .mcp.json
```

然后编辑 `.mcp.json`：

```json
{
  "mcpServers": {
    "xiaohongshu-mcp": {
      "type": "http",
      "url": "http://localhost:18060/mcp"
    },
    "amap-maps": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@amap/amap-maps-mcp-server"
      ],
      "env": {
        "AMAP_MAPS_API_KEY": "YOUR_AMAP_MAPS_API_KEY"
      }
    },
    "notion": {
      "command": "npx",
      "args": [
        "-y",
        "@notionhq/notion-mcp-server"
      ],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer YOUR_NOTION_TOKEN\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

需要替换的值：

- `YOUR_AMAP_MAPS_API_KEY`：高德地图 API Key。
- `YOUR_NOTION_TOKEN`：Notion Integration Token，通常以 `ntn_` 或 `secret_` 开头。
- `xiaohongshu-mcp.url`：小红书 MCP 的本地地址，默认是 `http://localhost:18060/mcp`。

## 小红书 MCP

启动方式取决于团队使用的小红书 MCP 实现。本项目只约定它通过 HTTP MCP 暴露在：

```text
http://localhost:18060/mcp
```

首次使用通常需要完成登录或扫码授权。启动后再打开 Cursor 或重载 MCP 配置。
如果本地使用仓库内的小红书 MCP 二进制文件，连接失败或首次登录时运行：

```bash
./xiaohongshu/xiaohongshu-mcp-darwin-arm64 -headless=false
```

保持该进程运行后，再重试小红书 MCP 工具调用。

## 高德地图 MCP

高德地图 MCP 由 Cursor 按 `.mcp.json` 中的配置通过 `npx` 启动：

```json
{
  "command": "npx",
  "args": ["-y", "@amap/amap-maps-mcp-server"]
}
```

团队成员只需要在 `.mcp.json` 中填入自己的 `AMAP_MAPS_API_KEY`。

## Notion MCP

Notion MCP 同样由 Cursor 按 `.mcp.json` 中的配置通过 `npx` 启动：

```json
{
  "command": "npx",
  "args": ["-y", "@notionhq/notion-mcp-server"]
}
```

Notion Integration 需要被授权访问目标 Database 或父页面，否则后续写入页面会失败。

`OPENAPI_MCP_HEADERS` 必须是一个 JSON 字符串，例如：

```json
"{\"Authorization\": \"Bearer YOUR_NOTION_TOKEN\", \"Notion-Version\": \"2022-06-28\"}"
```

## Skill 调用 `flyai-cli` 的约束

为了保证 Skill 可以被团队和 GitHub 复用，飞猪能力只通过命令行调用，不依赖本机已有的 `flyai` skill。

推荐在 Skill 中直接使用这些命令：

```bash
flyai keyword-search --query "杭州 3日游"
flyai search-hotel --dest-name "杭州" --hotel-types "高端酒店" --check-in-date "2026-05-01" --check-out-date "2026-05-03"
flyai search-flight --origin "北京" --destination "杭州" --dep-date "2026-05-01"
flyai search-poi --city-name "杭州" --keyword "西湖 门票"
```

注意参数名必须使用 `flyai-cli` 的实际参数：

- 酒店：`--dest-name`、`--hotel-types`、`--check-in-date`、`--check-out-date`
- 机票：`--origin`、`--destination`、`--dep-date`
- 景点：`--city-name`、`--keyword`

## 验证清单

配置完成后，按顺序验证：

```bash
python -m json.tool .mcp.json >/dev/null
flyai --help
flyai keyword-search --query "Shanghai cruise"
```

如果 `flyai keyword-search` 返回 `status: 0` 和 `message: success`，说明 CLI 可用。MCP 是否可用需要在 Cursor 中重载项目后检查 MCP 工具列表，或直接让 Agent 调用对应 MCP 工具验证。

## 提交到 GitHub 前

可以提交：

- `README.md`
- `.mcp.example.json`
- `.claude/skills/` 中后续设计完成的 Skill 文件

不要提交：

- `.mcp.json`
- Notion Token
- 高德地图 API Key
- 本地登录态、缓存或用户个人旅行档案
