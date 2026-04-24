# 第四阶段：生成 Notion 攻略报告

## 核心原则

由于 Notion MCP 需要重启 Claude Code 才能生效，我们使用 **Python urllib 直接调用 Notion REST API**。

---

## 1. Notion API 配置

### 1.1 读取 Token

从 `~/.claude.json` 读取 Notion Token：

```python
import json

with open("/Users/q/.claude.json") as f:
    config = json.load(f)

# 从 OPENAPI_MCP_HEADERS 中提取 Token
headers_str = config["mcpServers"]["notion"]["env"]["OPENAPI_MCP_HEADERS"]
headers = json.loads(headers_str)
TOKEN = headers["Authorization"].replace("Bearer ", "")
NOTION_VERSION = headers["Notion-Version"]
```

### 1.2 API Headers

```python
headers = {
    "Authorization": f"Bearer {TOKEN}",
    "Notion-Version": "2022-06-28",
    "Content-Type": "application/json"
}
```

---

## 2. Database 初始化

### 2.1 搜索已有 Database

```python
import urllib.request, json

data = json.dumps({
    "query": "旅游攻略库",
    "filter": {"value": "database", "property": "object"}
}).encode()

req = urllib.request.Request(
    "https://api.notion.com/v1/search",
    data=data,
    headers=headers,
    method="POST"
)

with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())

if result["results"]:
    db_id = result["results"][0]["id"]
    print(f"Found existing DB: {db_id}")
else:
    print("No existing DB, need to create")
```

### 2.2 创建新 Database

若不存在，创建新 Database：

```python
# 先找到 workspace 根页面
data = json.dumps({
    "filter": {"value": "page", "property": "object"},
    "page_size": 5
}).encode()

req = urllib.request.Request(
    "https://api.notion.com/v1/search",
    data=data,
    headers=headers,
    method="POST"
)

with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())

# 找到 workspace 类型的 parent
workspace_page_id = None
for page in result["results"]:
    if page.get("parent", {}).get("type") == "workspace":
        workspace_page_id = page["id"]
        break

# 创建 Database
db_payload = {
    "parent": {"type": "page_id", "page_id": workspace_page_id},
    "icon": {"type": "emoji", "emoji": "🗺️"},
    "title": [{"type": "text", "text": {"content": "🗺️ 旅游攻略库"}}],
    "properties": {
        "Name": {"title": {}},
        "目的地": {"rich_text": {}},
        "出行日期": {"date": {}},
        "天数": {"number": {"format": "number"}},
        "人均预算": {"number": {"format": "number"}},
        "同行人": {"select": {"options": [
            {"name": "单人", "color": "blue"},
            {"name": "情侣", "color": "pink"},
            {"name": "家庭", "color": "green"},
            {"name": "朋友", "color": "orange"}
        ]}},
        "节奏": {"select": {"options": [
            {"name": "特种兵", "color": "red"},
            {"name": "适中", "color": "yellow"},
            {"name": "度假", "color": "green"}
        ]}},
        "状态": {"select": {"options": [
            {"name": "规划中", "color": "blue"},
            {"name": "已完成", "color": "green"},
            {"name": "收藏", "color": "pink"}
        ]}}
    }
}

data = json.dumps(db_payload).encode()
req = urllib.request.Request(
    "https://api.notion.com/v1/databases",
    data=data,
    headers=headers,
    method="POST"
)

with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
    db_id = result["id"]
```

---

## 3. 创建攻略页面

### 3.1 Block 构建辅助函数

```python
def txt(content, bold=False, link=None):
    t = {"type": "text", "text": {"content": content}}
    if link:
        t["text"]["link"] = {"url": link}
    if bold:
        t["annotations"] = {"bold": True}
    return t

def h1(content):
    return {"object":"block","type":"heading_1","heading_1":{"rich_text":[txt(content)]}}

def h2(content):
    return {"object":"block","type":"heading_2","heading_2":{"rich_text":[txt(content)]}}

def h3(content):
    return {"object":"block","type":"heading_3","heading_3":{"rich_text":[txt(content)]}}

def para(*parts):
    return {"object":"block","type":"paragraph","paragraph":{"rich_text":list(parts)}}

def divider():
    return {"object":"block","type":"divider","divider":{}}

def bullet(*parts):
    return {"object":"block","type":"bulleted_list_item","bulleted_list_item":{"rich_text":list(parts)}}

def todo(content, checked=False):
    return {"object":"block","type":"to_do","to_do":{"rich_text":[txt(content)],"checked":checked}}

def callout(content, emoji="📍"):
    return {"object":"block","type":"callout","callout":{"rich_text":[txt(content)],"icon":{"type":"emoji","emoji":emoji}}}

def table_row(cells):
    return {"object":"block","type":"table_row","table_row":{"cells":[[txt(c)] for c in cells]}}

def image_block(url):
    return {"object":"block","type":"image","image":{"type":"external","external":{"url":url}}}
```

### 3.2 创建页面（第一批 blocks）

⚠️ **重要限制**：Notion API 单次请求最多 100 个 blocks。

```python
# 构建前 95 个 blocks（留 5 个余量）
blocks_batch1 = [
    image_block(COVER_IMG),
    divider(),
    h2("📊 行程总览"),
    # ... 更多 blocks
]

page_payload = {
    "parent": {"type": "database_id", "database_id": db_id},
    "icon": {"type": "emoji", "emoji": "🏯"},
    "cover": {"type": "external", "external": {"url": COVER_IMG}},
    "properties": {
        "Name": {"title": [{"type": "text", "text": {"content": "成都3日游完全攻略 | 2026年五一"}}]},
        "目的地": {"rich_text": [{"type": "text", "text": {"content": "成都"}}]},
        "出行日期": {"date": {"start": "2026-05-01", "end": "2026-05-03"}},
        "天数": {"number": 3},
        "人均预算": {"number": 1685},
        "同行人": {"select": {"name": "朋友"}},
        "节奏": {"select": {"name": "适中"}},
        "状态": {"select": {"name": "规划中"}}
    },
    "children": blocks_batch1
}

data = json.dumps(page_payload, ensure_ascii=False).encode("utf-8")
req = urllib.request.Request(
    "https://api.notion.com/v1/pages",
    data=data,
    headers=headers,
    method="POST"
)

with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
    page_id = result["id"]
    page_url = result["url"]
```

### 3.3 追加剩余 blocks

```python
# 构建剩余 blocks
blocks_batch2 = [
    h3("Day 3 — ..."),
    # ... 更多 blocks
]

data = json.dumps({"children": blocks_batch2}, ensure_ascii=False).encode("utf-8")
url = f"https://api.notion.com/v1/blocks/{page_id}/children"
req = urllib.request.Request(url, data=data, headers=headers, method="PATCH")

with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
```

---

## 4. Table 创建

### 4.1 Table 结构

```python
{
    "object": "block",
    "type": "table",
    "table": {
        "table_width": 2,  # 列数
        "has_column_header": True,  # 是否有表头
        "has_row_header": False,  # 是否有行头
        "children": [
            table_row(["列1", "列2"]),
            table_row(["数据1", "数据2"]),
        ]
    }
}
```

### 4.2 示例：行程总览表

```python
{
    "object": "block",
    "type": "table",
    "table": {
        "table_width": 2,
        "has_column_header": False,
        "has_row_header": True,
        "children": [
            table_row(["目的地", "成都 🏯"]),
            table_row(["出行天数", "3天2晚"]),
            table_row(["出行日期", "2026-05-01 至 2026-05-03"]),
            table_row(["同行人", "好朋友（2人）"]),
            table_row(["人均预算", "经济型（< ¥500/日）"]),
        ]
    }
}
```

---

## 5. 错误处理

### 5.1 常见错误

#### 错误 1：超过 100 blocks 限制

```
body.children.length should be ≤ 100, instead was 112
```

**解决**：分批创建，第一批 ≤ 95 个，剩余用 PATCH 追加。

#### 错误 2：Token 无效

```
401 Unauthorized
```

**解决**：检查 `~/.claude.json` 中的 Token 是否正确。

#### 错误 3：Database 不存在

```
404 Not Found
```

**解决**：先搜索 Database，若不存在则创建。

### 5.2 错误处理模板

```python
try:
    with urllib.request.urlopen(req) as resp:
        result = json.loads(resp.read())
except urllib.error.HTTPError as e:
    body = e.read().decode()
    print(f"HTTP Error {e.code}: {body[:2000]}")
    # 根据错误码处理
```

---

## 6. 完整页面结构

### 页面内容顺序

1. **封面图片**（从小红书获取）
2. **行程总览表**（目的地、天数、日期、预算等）
3. **天气提示 & 预约提示**（callout）
4. **大交通推荐**（航班/高铁，含飞猪链接）
5. **每日行程**（Day 1, Day 2, Day 3）
   - 上午、午餐、下午、晚餐、夜间、住宿
6. **住宿精选**（2-3 家推荐，含飞猪链接）
7. **门票 & 预约指南**（表格）
8. **详细预算清单**（表格）
9. **避坑指南**（callout）
10. **行前准备清单**（todo list）
11. **参考攻略来源**（小红书链接）

---

## 7. 性能优化

### 7.1 批量处理

- 第一批：封面 + 总览 + 大交通 + Day1 + Day2（约 70 blocks）
- 第二批：Day3 + 住宿 + 门票 + 预算 + 避坑 + 清单（约 50 blocks）

### 7.2 并行创建

若有多个攻略需要创建，可以并行发起请求（但注意 API 速率限制）。

---

## 8. 常见问题

### Q: 如何设置 Gallery View？
A: Gallery View 需要通过 Notion UI 手动设置，API 暂不支持创建 View。创建 Database 后，用户可在 Notion 中手动添加 Gallery View。

### Q: 图片无法显示？
A: 检查图片 URL 是否可访问，小红书图片 URL 有时效性。

### Q: Table 显示错乱？
A: 检查 `table_width` 是否与 `table_row` 的 cells 数量一致。

### Q: 如何更新已有页面？
A: 使用 PATCH 方法更新页面属性或追加 blocks：
```python
url = f"https://api.notion.com/v1/pages/{page_id}"
# 或
url = f"https://api.notion.com/v1/blocks/{page_id}/children"
```

---

## 9. 完整示例代码

参考测试中的完整 Python 代码（见 `测试问题梳理.md` 中的实际执行记录）。
