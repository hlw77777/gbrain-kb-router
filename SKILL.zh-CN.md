---
name: kb-router
description: >-
  知识库问答三档路由（中文参考版）：L1 零成本轻查（search/recall/entity）、
  L2 零成本 harness 综合（默认：查询变体 + 并行检索 expand:false + takes/图检索
  + harness 终稿 + rate_answer 喂反馈）、L3 付费 think 重炮（教义级内容铁律 /
  L2 两轮仍稀薄 / 重要决策级综合 / 用户点名 think）。
  Triggers on: 查库、知识库问题、深答、梳理、对比、溯源、跨页综合、think 路由、
  kb-router、Gbrain think、走 L3。
---

# 知识库问答路由（kb-router · 中文参考版）

> 原则：**harness 综合为主力（免费且终稿质量高），think 管线为重炮（按次付费，只留给真正需要的问题）。**
> 英文主版本见 [SKILL.md](SKILL.md)；本文件是原始 doctrine 的脱敏参考版。

## 路由口诀

**L1 管点，L2 管面（默认），L3 管教义和硬骨头。**

## 第 0 步：分类（<10 秒）

| 判据 | 路由 |
|---|---|
| 纯精确查找（文件在哪/单一事实/实体卡片） | L1 |
| 跨页梳理、对比、溯源、why/how，非教义级 | L2（默认） |
| 教义级语料问题（有锚点地图的课程/规范内容） | L3（铁律，`anchor:"sources/<你的语料地图>"`） |
| 用户明说「Gbrain think / 走 L3」 | L3（点名即重炮） |
| 用户当轮明说「快答/简短/一句话」 | L1 或最短路径 |
| 纯外部信息（不查库） | 不进本路由，走外部检索 |

## L1 轻查

`search`（精确 token/人名）→ `entity`（卡片）→ `recall`（事实）。问完即走。

## L2 harness 综合（默认档）

1. **免费复刻 expansion**：自己出 3-5 个查询变体（同义改写、上下游概念、中英混合），替代服务端付费扩展。
2. **并行检索（一律 `expand:false`）**：
   - `query`：`expand:false, return_unit:"section", token_budget:12000`（概念/梳理类变体）
   - `search`：精确 token 类变体
   - `takes_search`：问题涉及观点/判断/预测时
   - `traverse_graph`：有明确实体锚点时（`slug` + `depth:2`）
3. **读全段**：证据不足 `get_page`；凡要引用的原文，一律文件级实查验证行号。
4. **harness 终稿**：引用（slug/可点链接）齐全、缺口明说、冲突并列。
5. **喂反馈**：返回带 `answer_id` 时调 `rate_answer`（有用 4-5，无关 1-2）；远程 HTTP 通道无 answer_id 则跳过。

## L3 think 重炮（升级判据，任一命中）

1. **教义级语料**：一律 `think` + 锚点（把你的核心语料地图钉成一个 anchor）；引用原文必须文件二次验证。
2. **L2 两轮仍不达标**：证据稀薄、冲突无解、关键页检索不到。
3. **重要决策级综合**：值得喂「引用自动喂权重」通道的长期投资问题。
4. **用户点名 think**。

think 输出永远当**证据包+草稿**，终稿 harness 写（含 conflicts/gaps 的翻译与文件验证）。
L2 与 L3 打架时：文件实查原文为准——确定性大于任何模型综合。

**L3 前置健康检查**：think 返回 `synthesis_status` 非 ok（no LLM / 余额不足 / not_json）= 服务端 LLM 通道故障——立即降级按 L2 配方完成本题（think 的 gather 证据仍可当补充线索），并如实报告通道故障。

## 通道差异

| 项 | stdio / 本地 CLI | 远程 HTTP MCP |
|---|---|---|
| answer_id / rate_answer | 返回 → **必做** | 不返回 → 跳过 |
| 用户视角 takes（holder=self） | agent 通道可能检索不到 → Bash 走 owner CLI：`gbrain takes list --who self` | 同左 |

## 官方 brain-first 纪律（合并采纳）

- 检索链强制顺序：search（精确）→ query（概念）→ get_page → 外部兜底；查 vault 前先问 gbrain。
- score > 0.5 即用；notice 先读（degraded ≠ 没货）；[STALE] 必须提示。
- 用户原话最高权威（用户陈述 > compiled truth > timeline > 外部）。
- 付费外部拉取先 `capture` 入库，避免重复付费。
- 「search 有命中」不是覆盖证明；枚举用 `list_pages`，改写用 `query`。

## 失效边界

- 单一事实不需要变体扇面（L1 即可）。
- 纯外部问题不进本路由。
- 用户要简短时，跳过路由。
