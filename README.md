# TourFood - 小红书美食探索工具

使用 Playwright 在小红书上搜索城市美食和本地餐厅的 Agent Skill 合集。

## 前置准备

### 0. 为agent安装 Playwright 插件

## 使用流程

### Step 1：搜索城市美食

加载 `xiaohongshu_food_search.md` Skill，搜索目标城市的热门美食：

```
加载 xiaohongshu_food_search.md
```

然后告诉 Agent 要搜索的城市，例如：

```
帮我搜索广州的特色美食
```

Agent 会：
- 使用 2 个关键词搜索（城市美食攻略、特色美食推荐）
- 每个关键词选取 4 种热度级别的帖子
- 提取帖子内容和热门评论
- 保存到 `food_search_results/select_food.json`

### Step 2：搜索具体美食的本地餐厅

根据 Step 1 的美食推荐，加载 `xiaohongshu_restaurant_search.md` Skill，搜索具体美食的本地餐厅：

```
加载 xiaohongshu_restaurant_search.md
```

然后告诉 Agent 想吃的具体食物，例如：

```
帮我搜索郴州鱼粉的本地餐厅
```

Agent 会：
- 使用 1 个关键词搜索
- 选取 6 个帖子（2 个网红店 + 4 个本地人推荐）
- 深度分析评论区，识别本地人常去的餐厅
- 判断依据来自评论中的本地人推荐
- 保存到 `restaurant_search_results/select_restaurant.json`

## 输出文件

| 文件 | 说明 |
|------|------|
| `food_search_results/select_food.json` | 城市美食搜索结果 |
| `food_search_results/tab_records.json` | 美食搜索 Tab 记录 |
| `restaurant_search_results/select_restaurant.json` | 本地餐厅搜索结果 |
| `restaurant_search_results/tab_records.json` | 餐厅搜索 Tab 记录 |

## 文件说明

| 文件 | 用途 |
|------|------|
| `xiaohongshu_food_search.md` | 搜索城市热门美食的 Agent Skill |
| `xiaohongshu_restaurant_search.md` | 搜索具体食物本地餐厅的 Agent Skill |
| `PopFood_Search_Skill.md` | 原始版本（已弃用） |
