---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 37995
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

用 OpenClaw 做自动化一段时间后，多数人会撞上同一个矛盾：想让助手"什么都会"，又不希望它什么都带在身上。早期做法是把所有操作手册塞进 AGENTS.md：邮件怎么发、报表怎么跑……结果上下文动辄上万 token，注意力被稀释，反而哪件事都做不精。Skills 机制就是为此设计的：把能力拆成独立目录，**按需加载**。

## 它是怎么工作的

一个 Skill 是一个文件夹，核心是 `SKILL.md`：YAML frontmatter 写 name 和 description，正文写流程说明，旁边可带脚本、参考文档、模板。加载分三层（渐进式披露）：

1. **元信息常驻**：只有 name 和 description 长期占用上下文，每个 skill 约几十 token；
2. **正文按需**：任务命中某 skill 的 description 时，agent 才读完整 SKILL.md；
3. **资源再按需**：正文引用的长文档和脚本，真正执行时才读。

装 50 个 skill，常驻成本可能还不如一份大 prompt 的零头。

## 动手步骤

以"每周从数据库拉数据生成周报"为例：

1. 在 workspace 下建 `skills/weekly-report/`，写入 `SKILL.md`：

```markdown
---
name: weekly-report
description: Use when 用户要求生成周报、汇总本周数据时。
  涉及 SQL 导出、指标汇总、Markdown 报告输出。
---

# 周报生成

1. 运行 scripts/export.py 拉取本周数据
2. 按 references/template.md 的结构汇总
3. 输出到 workspace/reports/
```

2. 确定性逻辑写成脚本放进 `scripts/`，别用大段文字描述"如何写 SQL"——脚本一次执行，比模型每次重新推理更省也更稳；
3. 模板、字段说明等长文档放 `references/`，正文只留一行指引；
4. 开新会话（或重载 gateway），问一句相关任务，看日志里 skill 是否触发；
5. 没触发就改 description，触发太频繁也改 description。**description 就是 skill 的触发接口。**

## 踩坑点

- **description 写得太泛**：一句"处理数据"，什么任务都命中，正文被反复加载，等于退化回大 prompt；
- **正文过长**：几百行的 SKILL.md，触发一次就是一次上下文冲击，超过约 500 行就该拆 references；
- **脚本写死绝对路径**：跨机器同步即失效，统一用 workspace 相对路径；
- **把 skill 当插件用**：skill 本质是提示词层的知识，不是沙箱化的执行单元。需要系统集成、长驻进程、外部凭证管理，该上 MCP 或 plugin 就别硬写成 skill；
- **机密写进 SKILL.md**：skill 会被 git 同步、被分享，密钥一律走环境变量；
- **改完不重载**：当前会话不会热生效，容易误判成"没触发"。

## 可复用建议

- description 用固定句式：`Use when <场景>，尤其涉及 <关键词>`，比自由发挥命中率高得多；
- 一个 skill 只做一件事，小而组合胜过大而全；
- 确定性操作下沉到脚本，长文档拆到 references，正文只留判断和流程；
- 高度个人化的留在本地 workspace，通用的沉淀成可分享的 skill；
- 用 git 管 skills 目录，每次改 description 都留 commit，方便回溯"哪个版本触发了什么"。

## 总结

Skills 的价值不在"多"，在"准"：用极小的常驻成本，换取任务发生时的完整能力。写 skill 时把八成精力花在 description 上——它是 agent 决定是否加载你能力的唯一依据；剩下两成，用来把正文写短、把逻辑下沉到脚本。做到这两点，skill 体系才能随时间越长越稳，而不是越长越乱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/d0402de0d7cec4ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/6b5424edf058e994.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/1b5f83720f8f178d.png)

