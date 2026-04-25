---
name: travel-research
description: 为个性化旅游攻略采集证据和候选方案。需要搜索小红书真实笔记、高德地图天气/距离/路线、飞猪酒店机票门票时使用；只直接调用 flyai-cli，不依赖外部 flyai skill。
---

# Travel Research

## 目标

输出可用于排程的候选清单，而不是泛泛资料。每条关键推荐必须有来源、链接或高德计算依据。

## 输入

来自 `travel-intake` 的需求摘要，至少包含：

- 目的地、日期、出发/返程边界。
- 预算、节奏、同行人、兴趣、住宿和交通偏好。
- 饮食限制、避雷事项、特殊需求。

## 调研顺序

1. 前置检查：按 `references/00-tool-contracts.md` 验证 MCP 和 `flyai-cli`。
2. 大交通：按 `references/01-transport-boundary.md` 搜索候选，但不得替用户选择。
3. 景点活动：先用高德 POI/详情定位，再按 `references/02-attractions-activities.md` 补门票、避雷和图片。
4. 酒店餐饮：先用高德周边搜索围绕景区簇筛选位置，再按 `references/03-hotels-dining.md` 补价格、评价、链接。
5. 通勤天气风险：按 `references/04-routing-weather-risk.md` 用高德多方式路线计算、距离、天气和应急。
6. 证据整理：按 `references/05-evidence-output.md` 输出带来源的调研包。

## 必须产出

```markdown
## 调研结果
### 大交通候选
### 核心景点/活动候选
### 酒店候选
### 餐饮候选
### 关键通勤矩阵
### 图片素材
### 天气与室内备选
### 风险、预约与避雷
### 来源链接
```

## 证据规则

- 小红书笔记：记录标题、要点、链接、适用维度。
- 飞猪产品：只用于价格、预订、票务和产品链接，价格标注“参考价格，以实时为准”。
- 高德结果：记录 POI、地址、坐标、商圈、起终点、方式、距离、预计时间。
- 图片素材：记录对象、图片 URL、来源链接、用途建议；不可编造来源。
- 冲突信息优先级：官方/实时工具 > 高德路线 > 飞猪产品 > 小红书经验。

## Reference

- 工具契约：`references/00-tool-contracts.md`
- 大交通边界：`references/01-transport-boundary.md`
- 景点活动：`references/02-attractions-activities.md`
- 酒店餐饮：`references/03-hotels-dining.md`
- 通勤天气风险：`references/04-routing-weather-risk.md`
- 证据输出：`references/05-evidence-output.md`
