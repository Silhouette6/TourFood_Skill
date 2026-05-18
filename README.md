# TourFood - 小红书美食探索工具

基于全局 `xhs` CLI 的 Agent Skill 合集，用于搜索城市美食和具体食物的本地餐厅，并把帖子、评论和判断依据保存为结构化 JSON。

## 前置准备

### 0. 为agent安装 Playwright 插件

本项目依赖 [`jackwener/xiaohongshu-cli`](https://github.com/jackwener/xiaohongshu-cli) 提供的 `xhs` 命令。

推荐使用 `uv tool` 安装，速度快且环境隔离：

```powershell
uv tool install xiaohongshu-cli
```

也可以使用 `pipx` 安装：

```powershell
pipx install xiaohongshu-cli
```

升级到最新版本：

```powershell
uv tool upgrade xiaohongshu-cli
pipx upgrade xiaohongshu-cli
```

建议定期升级，避免因版本过旧导致 API 调用异常。

从源码安装：

```powershell
git clone git@github.com:jackwener/xiaohongshu-cli.git
cd xiaohongshu-cli
uv sync
```

确保本机已经可以直接调用 `xhs`：

```powershell
xhs login
xhs login --qrcode
```

本项目的 skill 不会在每次任务开始前主动检查登录状态。若搜索、读帖或评论命令提示未登录，再运行以上登录命令后重试。

## 常用 CLI 命令

本项目只依赖以下采集命令：

```powershell
xhs search "广州美食攻略" --page 1 --json
xhs read 1 --json
xhs comments 1 --json
xhs comments 1 --all --json
```

短索引 `1`、`2`、`3` 只对应最近一次列表命令的结果，例如最近一次 `xhs search ... --json`。处理当前搜索结果时，应先完成对应帖子的 `read` 和 `comments`，再执行新的搜索或翻页命令。

## 使用流程

### Step 1：搜索城市美食

加载 `xiaohongshu_food_search.md` Skill，搜索目标城市的热门美食：

```text
加载 xiaohongshu_food_search.md
```

然后告诉 Agent 要搜索的城市，例如：

```text
帮我搜索广州的特色美食
```

Agent 会：

- 使用 2 个固定关键词：`{城市}美食攻略`、`{城市}特色美食推荐`
- 每个关键词目标选取 4 篇帖子，合计最多 8 篇
- 使用 `xhs read --json` 读取帖子内容
- 使用 `xhs comments --json` 抓取评论，必要时补充 `--all`
- 每处理完一篇帖子就更新 `food_search_results/select_food.json`
- 生成 `food_search_results/command_records.json`
- 在最终总结中提炼推荐美食、代表店名或区域、评论依据

### Step 2：搜索具体美食的本地餐厅

根据 Step 1 的美食推荐，加载 `xiaohongshu_restaurant_search.md` Skill，搜索具体美食的本地餐厅：

```text
加载 xiaohongshu_restaurant_search.md
```

然后告诉 Agent 想吃的具体食物，例如：

```text
帮我搜索郴州鱼粉的本地餐厅
```

Agent 会：

- 使用 1 个关键词：`{食物名称}`
- 固定完成 6 篇帖子：2 篇高赞样本 + 4 篇中低赞样本
- 使用 `xhs read --json` 读取帖子内容
- 使用 `xhs comments --all --json` 全量抓取评论
- 基于点赞分层和评论证据区分本地人推荐餐厅与网红餐厅
- 每处理完一篇帖子就更新 `restaurant_search_results/select_restaurant.json`
- 生成 `restaurant_search_results/command_records.json`

## 输出文件

| 文件 | 说明 |
|------|------|
| `food_search_results/select_food.json` | 城市美食搜索结果 |
| `food_search_results/command_records.json` | 城市美食搜索 CLI 命令记录 |
| `restaurant_search_results/select_restaurant.json` | 本地餐厅搜索结果 |
| `restaurant_search_results/command_records.json` | 本地餐厅搜索 CLI 命令记录 |

`command_records.json` 只保存命令元数据，例如命令文本、关键词、页码、短索引、关联帖子和执行时间，不保存完整 CLI 输出。

## Skill 文件

| 文件 | 用途 |
|------|------|
| `xiaohongshu_food_search.md` | 搜索城市热门美食的 Agent Skill |
| `xiaohongshu_restaurant_search.md` | 搜索具体食物本地餐厅的 Agent Skill |

## 弃用文件

| 文件 | 说明 |
|------|------|
| `food_search_results/tab_records.json` | 旧 Playwright 流程的历史记录文件，新流程不再生成 |
| `restaurant_search_results/tab_records.json` | 旧 Playwright 流程的历史记录文件，新流程不再生成 |
