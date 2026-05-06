---
name: xiaohongshu-food-search
description: 使用 Playwright 在小红书搜索城市美食并提取帖子评论
---

# 小红书美食搜索 Skill

## 概述

使用 Playwright 在小红书搜索指定城市的美食内容，抓取帖子及评论，并结构化保存结果。

## 核心目标

对一个城市执行：

- 2 个关键词搜索
- 每个关键词选取 4 种热度帖子
- 共计 8 个帖子
- 输出结构化 JSON + Tab 记录

## 执行流程（强制顺序）

1. 启动浏览器
2. 检查登录状态（必要时暂停等待扫码）
3. 按关键词逐一搜索（ **强制** 每个关键词一个 Tab）
4. 每个关键词选取 4 个不同热度帖子（每个帖子一个 Tab）
5. 提取帖子内容 + 评论
6. 汇总数据并保存文件
7. 对最终保存文件进行简要总结汇报

## 关键词规则（固定）
为了获得更全面、多样化的美食信息，搜索必须使用以下 2 个关键词：

1. `{城市名}美食攻略`（如"广州美食攻略"）
2. `{城市名}特色美食推荐`（如"广州特色美食推荐"）

URL 方式搜索示例：
https://www.xiaohongshu.com/search_result?keyword=广州美食攻略
https://www.xiaohongshu.com/search_result?keyword=广州特色美食推荐

## 帖子选择规则（每个关键词 4 个）

在搜索结果中按点赞数划分：

| 类型 | 选择标准 |
|------|----------|
| 高赞 | 点赞数最高 |
| 中高赞 | 中等偏上 |
| 中低赞 | 中等偏下 |
| 低赞 | 点赞较少但有评论 |

## 数据提取字段（每个帖子）

访问帖子后立即把信息以json格式追加到food_search_results/select_food.json中，必须包含：
**标题、挑选的热门的评论**

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
  "comments": [],
  "url": ""
}
```

## 输出文件

### 1. 帖子结果

路径：`food_search_results/select_food.json`

```json
{
  "city": "",
  "keywords_used": [],
  "total_posts": 12,
  "posts": []
}
```

### 2. Tab 记录

路径：`food_search_results/tab_records.json`

```json
{
  "tabs": [
    {
      "tab_id": "Tab-1",
      "type": "search",
      "keyword": "",
      "url": "",
      "opened_at": ""
    },
    {
      "tab_id": "Tab-2",
      "type": "post",
      "title": "",
      "url": "",
      "keyword_source": "",
      "engagement_level": "",
      "opened_at": ""
    }
  ]
}
```

## Tab 管理规范

### 基本原则

- 所有操作必须在"新 Tab"中进行，禁止复用 Tab
- 搜索 Tab：每个关键词 → 新建 Tab，不关闭
- 帖子 Tab：每个帖子 → 新建 Tab（Ctrl/Cmd + 点击），不关闭

### 禁止行为（硬性）

- 禁止复用 Tab 搜索多个关键词
- 禁止关闭任何 Tab
- 禁止在帖子页返回继续操作
- 禁止直接导航覆盖当前页面

### 登录处理

若出现登录弹窗：
1. 立即停止操作
2. 提示用户扫码
3. 等待用户明确回复后继续

## 完成标准（Checklist）

- [ ] 已搜索 3 个关键词
- [ ] 每个关键词获取 4 个帖子
- [ ] 共 12 个帖子
- [ ] 所有字段完整
- [ ] JSON 文件生成成功
- [ ] Tab 记录完整
- [ ] 简要总结推荐结果
