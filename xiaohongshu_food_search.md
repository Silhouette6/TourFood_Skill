---
name: xiaohongshu-food-search
description: 使用全局 xhs CLI 在小红书搜索城市美食并结构化提取帖子与评论
---

# 小红书美食搜索 Skill

## 概述

使用全局已安装的 `xhs` CLI 搜索指定城市的美食内容，读取帖子和评论，并结构化保存结果。

本 skill 不做任务前登录校验。直接执行搜索；如果 `xhs` 命令提示未登录或登录失效，再提示用户运行 `xhs login` 或 `xhs login --qrcode` 后重试。

## 核心目标

对一个城市执行：

- 固定使用 2 个关键词搜索
- 每个关键词目标选择 4 篇帖子
- 总目标最多 8 篇帖子
- 不强制去重，重复帖子按实际处理条目计数
- 每处理完一篇帖子立即写入结果 JSON
- 输出结构化结果 JSON + CLI 命令记录

## 执行流程（强制顺序）

1. 根据城市名生成 2 个固定关键词。
2. 对第 1 个关键词执行 `xhs search "{keyword}" --page 1 --json`。
3. 从当前搜索结果里选择候选帖子，并立刻使用短索引读取帖子与评论。
4. 每完成一篇帖子，立即更新 `food_search_results/select_food.json`。
5. 如果当前页无法满足该关键词 4 篇目标，继续执行 `--page 2`、最多到 `--page 3` 补样。
6. 完成第 1 个关键词后，再处理第 2 个关键词，流程相同。
7. 写入 `food_search_results/command_records.json`，只记录命令元数据。
8. 最终基于帖子证据总结推荐美食、代表店名或区域、评论依据。

## CLI 短索引规则（关键）

`xhs read 1 --json` 和 `xhs comments 1 --json` 中的短索引只对应最近一次列表命令，例如最近一次 `xhs search ... --json` 的结果。

因此必须遵守：

- 每次 `xhs search ... --json` 后，先完成该页候选帖子的 `read` 和 `comments`。
- 不要在处理当前页短索引之前执行新的 `xhs search ... --json`、`xhs feed`、`xhs hot` 等列表命令。
- 如果后续还需要引用某篇帖子，必须把 `note_id` 或完整 `url` 写入结果 JSON。

## 关键词规则（固定）

为了获得更全面、多样化的城市美食信息，必须使用以下 2 个关键词：

1. `{城市名}美食攻略`，例如 `广州美食攻略`
2. `{城市名}特色美食推荐`，例如 `广州特色美食推荐`

搜索命令示例：

```powershell
xhs search "广州美食攻略" --page 1 --json
xhs search "广州特色美食推荐" --page 1 --json
```

默认使用 CLI 综合排序，不额外添加排序参数。

## 帖子选择规则（每个关键词目标 4 篇）

在同一关键词的搜索结果中尽量覆盖 4 种热度：

| 类型 | 选择标准 |
|------|----------|
| 高赞 | 当前结果中点赞数最高 |
| 中高赞 | 点赞数中等偏上，内容完整 |
| 中低赞 | 点赞数中等偏下，有可读评论 |
| 低赞 | 点赞较少但有评论或具体店名信息 |

如果第一页不足以选满 4 篇，可继续查 `--page 2` 和 `--page 3`。超过第 3 页仍不足时，保留已完成结果，并在最终汇报中说明不足原因。

## 采集命令

每篇候选帖子按以下顺序执行：

```powershell
xhs read 1 --json
xhs comments 1 --json
```

如果默认评论结果明显不足以判断美食价值，可补充执行：

```powershell
xhs comments 1 --all --json
```

若单篇帖子读取或评论失败，跳过该帖，并从当前页或后续页继续找替补候选。

## 数据提取字段（每个帖子）

每完成一篇帖子的 `read` 和 `comments` 后，立即追加或更新到 `food_search_results/select_food.json`。

```json
{
  "keyword": "",
  "engagement_level": "",
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
  "url": ""
}
```

评论优先保留对美食判断有用的信息：具体菜品、店名、街区、价格、排队体验、踩雷对比、本地人补充。

## 输出文件

### 1. 帖子结果

路径：`food_search_results/select_food.json`

```json
{
  "city": "",
  "keywords_used": [],
  "total_posts": 0,
  "posts": []
}
```

`total_posts` 写实际完成数量，最大目标为 8。

### 2. CLI 命令记录

路径：`food_search_results/command_records.json`

只保存命令元数据，不保存完整 CLI 输出。

```json
{
  "commands": [
    {
      "step_id": "search-1",
      "type": "search",
      "command": "xhs search \"广州美食攻略\" --page 1 --json",
      "keyword": "广州美食攻略",
      "page": 1,
      "executed_at": ""
    },
    {
      "step_id": "read-1",
      "type": "read",
      "command": "xhs read 1 --json",
      "keyword_source": "广州美食攻略",
      "selected_index": 1,
      "engagement_level": "高赞",
      "note_id": "",
      "note_url": "",
      "executed_at": ""
    },
    {
      "step_id": "comments-1",
      "type": "comments",
      "command": "xhs comments 1 --json",
      "keyword_source": "广州美食攻略",
      "selected_index": 1,
      "comments_fetched": 0,
      "executed_at": ""
    }
  ]
}
```

## 失败处理

- 搜索命令提示未登录或登录失效时，提示用户执行 `xhs login` 或 `xhs login --qrcode`。
- 单篇 `read` 或 `comments` 失败时，记录失败原因，跳过该帖并继续找替补。
- 超过第 3 页仍无法满足目标数量时，停止补样，并在最终汇报中说明缺口。

## 完成标准（Checklist）

- [ ] 已使用 2 个固定关键词
- [ ] 每个关键词目标处理 4 篇帖子，合计最多 8 篇
- [ ] 所有采集命令都使用 `--json`
- [ ] 每页搜索后先完成该页短索引处理，再进入下一页或下一关键词
- [ ] 每篇帖子处理后已立即更新 `select_food.json`
- [ ] 已生成或更新 `command_records.json`
- [ ] 最终总结包含推荐美食、代表店名或区域、评论依据
