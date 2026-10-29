---
title: OpenClaw Skills 机制拆解：让 Agent 按需加载能力
feedId: 42957
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

OpenClaw 的 Agent 由系统提示词 + 工具集驱动。随着接入的 MCP server、自动化脚本越来越多，常见的做法是把所有操作说明堆进 AGENTS.md 或系统提示词——结果是上下文里常年躺着几千 token 的"说明书"，实际用到的可能不到十分之一。

Skills 机制就是针对这个问题的：把能力拆成独立单元，平时只暴露一句话摘要，模型判断需要时才加载完整内容。

## 问题：上下文预算怎么花

全量注入有两个代价：token 开销线性增长；更隐蔽的是注意力被无关指令稀释，工具选择准确率反而下降。Skills 的核心思路是渐进式披露：

- 系统提示词只注入每个 skill 的 `name` + `description`，单个几十 token；
- 完整的 SKILL.md 正文、附带的脚本和参考文档，只在模型判断"这个场景需要"时才通过 read 工具加载。

## 做法

以新增一个"日报生成" skill 为例：

1. 建目录 `workspace/skills/daily-report/`；
2. 写 SKILL.md，frontmatter 至少包含 name 和 description：

```markdown
---
name: daily-report
description: 汇总当天 commit、issue 和群消息，生成 Markdown 日报。用户说"出日报"或"今日总结"时使用。
---

（正文：数据来源、步骤、输出模板）
```

3. 依赖外部命令或环境变量的，在 frontmatter 声明 `requires`（bins / env / config）。gateway 启动时会校验，缺依赖就标记为不可用，而不是运行到一半报错；
4. 重启 gateway，或跑 `openclaw doctor` 确认 skill 已进入索引。

关键认知：**description 是模型决定是否加载的唯一依据**。写"处理各种报告相关任务"等于没写，要写清触发词和边界条件。

## 踩坑点

- **description 太宽泛**：什么都沾边，导致不相关对话也触发加载，反向污染上下文。宁可写窄；
- **指望模型猜约定**：正文必须写明输入从哪来、输出到哪去、格式长什么样，模型不会自动补全口头约定；
- **没声明 requires**：脚本依赖 jq，目标机器没有，跑到一半失败。声明后 gateway 会提前拦下；
- **堆数量**：索引里挂几十个 skill，摘要本身也开始吃预算。用 config 定期禁用不活跃项；
- **改了 SKILL.md 忘了重启**：测试的永远是旧版本，排查半天其实是缓存。

## 可复用建议

- 一个 skill 只做一件事；复杂流程拆成多个，在正文里互相引用，而不是写成千行大杂烩；
- 附带脚本放进 skill 自己的目录（如 `scripts/`），正文里用相对路径引用，整个目录拷走即可复用；
- 用 git 管理 skills 目录。团队共享时，review 对象就是 description 和正文，比口头传约定可靠得多；
- 新增后先跑 `openclaw doctor`，加载状态和依赖检查都绿再进日常流程。

## 总结

Skills 的价值不在"能加功能"，而在控制上下文的边际成本：接入的能力越多，系统提示词反而可以保持很小。实践中把 description 当接口文档写，把 SKILL.md 当 runbook 写，模型的触发准确率和 token 开销都会有可感知的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/87dc159ca969592f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/64134414cc322972.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/bbfd9bc530ae4f51.png)

