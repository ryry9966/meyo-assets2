---
title: 从零搭建 OpenClaw 社区发帖机器人：架构设计与踩坑实录
feedId: 40194
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

社区的干货长期散落在三处：Git 提交记录、群聊里的问答、issue 区的讨论。每周精选帖靠人工整理，平均要花两三个小时，还经常漏掉好内容。于是我用 OpenClaw 搭了一个发帖机器人——不是让 AI 自由发挥写文章，而是搭一条「采集 → 草稿 → 人工审核 → 发布」的流水线。

## 问题拆解

真正要解的是三件事：

1. **信息聚合**：多个异构数据源，格式不一；
2. **重复劳动**：摘要、排版、发布动作高度模板化；
3. **发布一致性**：人工操作容易漏发、错发、重发。

## 架构与实现

整体分五层，每层职责单一：

- **采集层**：两个自定义 MCP 工具。`fetch_git_log` 直接 subprocess 调 git，`fetch_issues` 走社区 API 拉近期 issue。
- **调度层**：OpenClaw cron，每周一 09:00 触发，**每次运行使用独立 session**（这点关键，踩坑部分细说）。
- **生成层**：system prompt 把模型定位成「编辑」而非「作者」；模板 `digest-template.md` 和风格指南 `style.md` 放在 workspace，随上下文注入。
- **审核层**：草稿先发到我的私聊窗口，回复确认后才允许调用发布工具。
- **发布层**：MCP 工具 `publish_post` 调论坛 API，成功后把 post_id 和内容 hash 写入 `state.json`。

## 踩坑实录

**1. 重发事故。** 首次发布超时后重试，同一个帖子发了两遍。解法：发布前先在状态文件写入 intent 记录（内容 hash + 时间戳），成功后再更新结果；启动时发现「有 intent 无结果」就跳过本轮并告警。

```json
{ "hash": "a3f2…", "intent": true, "post_id": null }
```

**2. 模型编造链接。** 草稿里出现了不存在的 issue 编号。改法：prompt 明确禁止生成上下文中没有的 URL 和编号，发布前用正则校验所有链接必须来自采集结果。

**3. session 污染。** 最初 cron 复用主 session，跑了几周后上下文膨胀、文风漂移。改成每次运行独立 session 后才稳定。

**4. 沙箱出网被拦。** `fetch_issues` 长期超时，排查半天发现是沙箱默认禁外网，给 MCP 工具进程加了论坛 API 域名白名单才通。

**5. Markdown 方言。** 论坛不认部分语法，嵌套代码围栏被截断。模板里固定围栏符号，发布前加一道转义检查。

## 可复用建议

- **模型只做格式化，不做事实来源**：所有数字、链接、结论必须来自工具返回值；
- **状态落盘**：一个 JSON 文件 + 原子写，胜过任何内存方案；
- **前几周保留人工审核**，稳定后降级为抽检，别一步到位；
- **运行日志写成 markdown 存进 workspace**：agent 自己能读历史，排查文风漂移时非常有用。

## 总结

发帖机器人本身不难，难的是把「发布」变成一条可审计、可回滚的流水线。OpenClaw 的 cron + MCP + workspace 组合刚好覆盖调度、工具、上下文三块，剩下的幂等、校验、审核都是通用工程解法。这套结构同样适用于周报、变更通告等任何「定期从数据源生成内容并发布」的场景。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/1f307af46e2eda56.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/848283c5affc8c54.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/d13956939d09cfeb.png)

