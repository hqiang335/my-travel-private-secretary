# 工具契约

## flyai-cli

只直接调用 CLI，不读取或依赖外部 `flyai` skill。

常用命令：

```bash
flyai keyword-search --query "杭州 3日游"
flyai search-hotel --dest-name "杭州" --hotel-types "高端酒店" --check-in-date "2026-05-01" --check-out-date "2026-05-03"
flyai search-flight --origin "北京" --destination "杭州" --dep-date "2026-05-01"
flyai search-poi --city-name "杭州" --keyword "西湖 门票"
```

参数名不得写错：

- 酒店：`--dest-name`、`--hotel-types`、`--check-in-date`、`--check-out-date`
- 机票：`--origin`、`--destination`、`--dep-date`
- 景点：`--city-name`、`--keyword`

解析 JSON 时优先保留：名称、价格、时间、`jumpUrl`、`detailUrl`、`picUrl`、`mainPic`、`systemMessage`。

## 小红书 MCP

连接失败或工具不可用时，在项目目录运行：
```bash
./xiaohongshu/xiaohongshu-mcp-darwin-arm64 -headless=false
```

首次使用或登录失效时必须使用 `-headless=false` 完成扫码/登录。启动后确认服务暴露在 `.mcp.json` 的 `http://localhost:18060/mcp`。

使用约束：

- 小红书只用于真实评价、避雷、图片、拍照点、排队体验和用户体验补充。
- 禁止并发调用 `search_feeds`，每次只搜索 1 个关键词。
- 禁止大量翻页；每个主题最多 1-2 次搜索。
- 同一目的地、同一关键词当天优先复用 `.claude/cache/xiaohongshu/` 下的缓存。
- 用户提供小红书链接时优先使用链接，不再对同一主题重复搜索。
- `search_feeds` 超时后等待并用更窄关键词重试 1 次；仍失败则停止小红书搜索，标记“待补查”，继续用高德和飞猪完成攻略骨架。

超时处理模板：

```markdown
- 小红书关键词：
- 状态：超时 / 已降级
- 重试关键词：
- 影响维度：
- 替代来源：高德 / 飞猪 / 用户提供链接
```

## 高德地图 MCP

高德是地点、路线、天气和通勤判断的主工具，不得用 `flyai-cli` 替代这些判断。

常用工具：

- `maps_geo`：地址转经纬度。
- `maps_regeocode`：经纬度转地址。
- `maps_text_search`：按关键词搜索 POI。
- `maps_around_search`：按中心点和半径搜索周边 POI。
- `maps_search_detail`：查询 POI 详情、地址、商圈、类型、坐标。
- `maps_weather`：城市天气和预报。
- `maps_distance`：测量两点距离和基础耗时；参数是 `origins`、`destination`、`type`。
- `maps_direction_transit_integrated`：公交/地铁/火车等公共交通路线；参数是 `origin`、`destination`、`city`、`cityd`。
- `maps_direction_driving`：驾车/打车路线。
- `maps_direction_walking`：步行路线。
- `maps_bicycling`：骑行路线。

每个入选景区、酒店、餐馆、机场/车站都必须尽量拿到高德 POI 或坐标。每条关键路线必须比较至少 2 种可行方式；城市公共交通可用时优先比较公共交通和打车。

## Notion MCP

研究阶段只确认目标 Database 可写；页面生成由 `travel-publisher` 处理。

## 链接与图片

- 酒店、票务、交通产品必须有跳转链接。
- 小红书核心参考必须有笔记链接。
- 图片优先级：小红书笔记图片 > 飞猪返回图片 > 公开网络图片。
- 链接或图片缺失时标注“需二次确认”，不得编造。
