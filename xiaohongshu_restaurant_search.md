---
name: xiaohongshu-restaurant-search
description: 使用全局 xhs CLI 搜索特定食物的本地餐厅，并基于评论判断本地人与网红推荐
---

# 小红书本地餐厅搜索 Skill

## 概述

针对某一种具体食物，例如 `郴州鱼粉`，使用全局已安装的 `xhs` CLI 搜索小红书帖子，通过帖子正文和评论分析识别：

- 本地人常去餐厅
- 网红餐厅或游客常见餐厅

本 skill 不做任务前登录校验。直接执行搜索；如果 `xhs` 命令提示未登录或登录失效，再提示用户运行 `xhs login` 或 `xhs login --qrcode` 后重试。

## 核心目标

对一个食物关键词执行：

- 仅使用 1 个关键词：`{食物名称}`
- 必须完成 6 篇帖子
- 样本目标为 2 篇高赞 + 4 篇中低赞
- 评论必须使用全量抓取
- 餐厅分类以点赞分层为主，再用评论证据校正
- 每处理完一篇帖子立即写入结果 JSON
- 输出结构化结果 JSON + CLI 命令记录

## 执行流程（强制顺序）

1. 使用食物名称生成唯一搜索关键词。
2. 执行 `xhs search "{食物名称}" --page 1 --json`。
3. 从当前搜索结果选择候选帖子，并立刻使用短索引读取帖子与全量评论。
4. 每完成一篇帖子，立即更新 `restaurant_search_results/select_restaurant.json`。
5. 如果当前页无法补满 6 篇，继续执行 `--page 2`、最多到 `--page 3`。
6. 单篇帖子读取或评论失败时跳过该帖，并继续找替补，直到补满 6 篇或已检查到第 3 页。
7. 写入 `restaurant_search_results/command_records.json`，只记录命令元数据。
8. 汇总本地人推荐餐厅、网红餐厅和判断依据。

## CLI 短索引规则（关键）

`xhs read 1 --json` 和 `xhs comments 1 --all --json` 中的短索引只对应最近一次列表命令，例如最近一次 `xhs search ... --json` 的结果。

因此必须遵守：

- 每次 `xhs search ... --json` 后，先完成该页候选帖子的 `read` 和 `comments`。
- 不要在处理当前页短索引之前执行新的 `xhs search ... --json`、`xhs feed`、`xhs hot` 等列表命令。
- 如果后续还需要引用某篇帖子，必须把 `note_id` 或完整 `url` 写入结果 JSON。

## 搜索关键词规则（固定）

只允许使用 1 个搜索关键词：`{食物名称}`。

示例：

- `郴州鱼粉`
- `柳州螺蛳粉`
- `顺德双皮奶`

搜索命令示例：

```powershell
xhs search "郴州鱼粉" --page 1 --json
```

默认使用 CLI 综合排序，不额外添加排序参数。

## 帖子选择规则（严格）

总计必须完成 6 篇帖子：

| 类型 | 数量 | 标准 |
|------|------|------|
| 高赞（网红候选） | 2 | 当前搜索结果中点赞数最高，通常作为网红或游客常见样本 |
| 中低赞（本地候选） | 4 | 点赞中低、有评论、出现具体店名或本地语境 |

若第 1 页不足，继续查 `--page 2` 和 `--page 3`。若单篇失败，跳过并补选下一篇；餐厅任务仍以完成 6 篇为标准。

## 采集命令

每篇候选帖子按以下顺序执行：

```powershell
xhs read 1 --json
xhs comments 1 --all --json
```

如果评论里出现关键回复线索，可按需追加：

```powershell
xhs sub-comments <note_id> <comment_id> --json
```

`comments --all --json` 是餐厅判断的默认要求，不使用默认评论页替代。

## 本地人与网红分类逻辑

分类以点赞分层为主：

- 高赞样本默认先归为网红或游客常见候选。
- 中低赞样本默认先归为本地候选。
- 读取评论后必须用证据校正分类。

强本地信号包括：

- 评论中对比多个本地店，例如 `这家对面更好吃`、`某某路那家才是天花板`。
- 出现本地语境，例如 `我们这边`、`本地人都去`、`从小吃到大`。
- 明确推荐非网红店，例如 `游客不会去这里`、`不是网红但好吃`。
- 提到具体街道、学校、市场、老城区或长期消费经验。

需要谨慎处理：

- 明显营销语气
- 只有打卡语气、缺少评论证据的高赞帖子
- 只有菜名或城市名、没有具体餐厅信息的内容

餐厅名称允许基于正文和评论做合理推断，但必须在 `local_analysis` 中说明依据。

## 数据提取字段（每个帖子）

每完成一篇帖子的 `read` 和 `comments --all` 后，立即追加或更新到 `restaurant_search_results/select_restaurant.json`。

```json
{
  "restaurant": [],
  "engagement_level": "",
  "classification": "local | tourist | uncertain",
  "title": "",
  "content": "",
  "author": "",
  "likes": 0,
  "favorites": 0,
  "comments_count": 0,
  "comments": [
    {
      "author": "",
      "content": "",
      "like_count": 0,
      "comment_id": "",
      "sub_comment_count": 0
    }
  ],
  "note_id": "",
  "url": "",
  "local_analysis": ""
}
```

`local_analysis` 必须引用正文或评论中的具体依据，说明餐厅名从哪里来、为什么归入本地或网红候选。

## 输出文件

### 1. 餐厅搜索结果

路径：`restaurant_search_results/select_restaurant.json`

```json
{
  "food": "",
  "total_posts": 6,
  "local_favorites": [],
  "tourist_favorites": [],
  "posts": []
}
```

`total_posts` 必须为 6，除非第 3 页以内无法完成且任务明确失败。

### 2. CLI 命令记录

路径：`restaurant_search_results/command_records.json`

只保存命令元数据，不保存完整 CLI 输出。

```json
{
  "commands": [
    {
      "step_id": "search-1",
      "type": "search",
      "command": "xhs search \"郴州鱼粉\" --page 1 --json",
      "keyword": "郴州鱼粉",
      "page": 1,
      "executed_at": ""
    },
    {
      "step_id": "read-1",
      "type": "read",
      "command": "xhs read 1 --json",
      "selected_index": 1,
      "engagement_level": "高赞",
      "note_id": "",
      "note_url": "",
      "executed_at": ""
    },
    {
      "step_id": "comments-1",
      "type": "comments",
      "command": "xhs comments 1 --all --json",
      "selected_index": 1,
      "comments_fetched": 0,
      "executed_at": ""
    }
  ]
}
```

## 失败处理

- 搜索命令提示未登录或登录失效时，提示用户执行 `xhs login` 或 `xhs login --qrcode`。
- 单篇 `read` 或 `comments --all` 失败时，记录失败原因，跳过该帖并继续找替补。
- 到第 3 页仍无法完成 6 篇时，停止任务并汇报缺口、已完成帖子和失败原因。

## 完成标准（Checklist）

- [ ] 已使用 1 个食物关键词
- [ ] 已完成 6 篇帖子
- [ ] 已包含 2 篇高赞样本和 4 篇中低赞样本
- [ ] 所有采集命令都使用 `--json`
- [ ] 每篇帖子都执行了 `xhs comments <index> --all --json`
- [ ] 每篇帖子处理后已立即更新 `select_restaurant.json`
- [ ] 已生成或更新 `command_records.json`
- [ ] 已区分本地人推荐餐厅和网红餐厅
- [ ] 最终推荐给出来自正文或评论的判断依据

## 最终输出要求

最终必须给出：

1. 本地人推荐餐厅
2. 网红餐厅或游客常见餐厅
3. 每个判断的证据来源，优先引用评论内容
4. 仍然不确定的餐厅及不确定原因
