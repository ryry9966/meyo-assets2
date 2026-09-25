---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 39033
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

Agent 接入日常自动化后，最常见的矛盾是：能力越来越多，上下文越来越贵。早期我把部署流程、告警处理、写作规范全塞进系统提示词，两千多行 prompt，模型注意力被稀释，简单任务也频繁跑偏。OpenClaw 的 Skills 机制就是针对这个问题的：把能力拆成独立的技能文件，平时只加载"目录"，命中时才展开"正文"。

## 渐进式加载的三级结构

1. **元数据常驻**：每个 skill 的 `name` 和 `description` 随系统提示词注入，只占几十个 token；
2. **正文按需**：模型判断当前任务命中某个 skill，才读取完整的 SKILL.md；
3. **资源再延迟**：SKILL.md 内可链接更长的参考文档、脚本，真正用到时才进上下文。

磁盘上一个 skill 就是一个目录，最少只需一个 SKILL.md：YAML frontmatter 写 `name`、`description`，正文写操作指引。位置放在 `~/.openclaw/skills`（用户级）或 workspace 的 `skills/`（工作区级）。管理用 `openclaw skills list` 看加载状态，`openclaw skills new` 生成模板。

## 实操步骤

1. 建目录、写 SKILL.md。**description 是触发关键，写"什么时候用"，不是"这是什么"**；
2. 正文控制在两百行内：步骤、约束、命令示例，不写散文；
3. 大篇幅参考资料拆成 `references/*.md`，正文只留路径引用；
4. `openclaw skills list` 确认元数据已注入；
5. 用真实任务验证：问一句和 skill 相关的话，观察模型是否去读正文。

最小示例：

```markdown
---
name: release-check
description: 发布前检查服务状态、日志和回滚预案时使用
---

# 发布前检查
1. 确认目标服务健康检查通过
2. 拉取最近 30 分钟错误日志并汇总
3. 输出回滚命令备用
```

## 踩坑点

- **description 写成名词解释**（"这是一个发布工具"），模型永远不会主动触发。要包含动词、场景、输入特征；
- **正文写成百科全书**，一次展开吃掉大量上下文，"按需加载"变成"按需爆炸"；
- **skill 引用的脚本没写清路径和依赖**，换台机器就断；
- **与 MCP 工具职责重叠**：同一件事既有 skill 又有 tool，模型随机选择，行为不稳定。我的原则是：流程性知识写成 skill，原子操作做成 tool；
- **改了不生效**：部分实现会缓存元数据，重载 gateway 或重开会话再验证，别急着怀疑模型。

## 可复用建议

- 把 description 当路由表维护，一句"在什么情况下读我"比十行说明有用；
- skills 目录进 git，技能迭代和代码一样走 review；
- 定期审计 `skills list`，长期没触发的 skill 下线或合并；
- 排障顺序固定：元数据注入了吗 → 正文被读取了吗 → 最后才考虑模型问题。

## 总结

Skills 的价值不是"更多能力"，而是"更干净的上下文"。默认少给、命中再给，触发的准确率和 token 成本都会改善。对已在用 MCP 的团队，建议先梳理能力清单：把流程迁移到 skill，把原子操作留在 tool，两边各司其职，比堆工具数量有效得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/38eeb51be2de5feb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/e8b175c6500553c0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1dc1763a15d56d2c.png)

