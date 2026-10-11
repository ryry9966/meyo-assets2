---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41190
source: 综合讨论
publishedAt: 2026-10-11
---

# 背景

做 Agent 的人迟早会撞上同一个矛盾：能力越多，上下文越贵。早期常见做法是把所有工具说明、插件文档、MCP server 的 schema 全部塞进 system prompt，结果 Agent 初始化就吃掉大几千 token，模型注意力被稀释，真正干活时反而容易选错工具。

OpenClaw 的 Skills 机制本质上是对这个问题的一个工程解法：**渐进式加载（progressive disclosure）**。启动时只注入每个技能的元数据（名称 + 一句话描述），正文和资源在模型判断"这个技能相关"时才被读进上下文。

# 问题拆解

在没有 Skills 的方案里，痛点集中在三处：

1. **常驻 token 成本**：十几个插件的文档全量常驻，多轮对话下成本线性上涨；
2. **工具选择质量下降**：工具/技能列表越长，误触发概率越高；
3. **能力无法独立演进**：能力和主 prompt 耦合，改一个插件描述就要动整个 prompt，没法单独测试和版本化。

# 做法

## 目录结构

一个技能就是一个目录，核心是 SKILL.md：

```text
skills/
  pdf-export/
    SKILL.md           # frontmatter + 指令正文
    scripts/render.py  # 确定性操作放脚本
    references/spec.md # 详细规范，按需读取
```

SKILL.md 的 frontmatter 只有两个关键字段：

```yaml
---
name: pdf-export
description: 将 Markdown 报告导出为 PDF。当用户要求"导出""生成 PDF""打印版"时触发。
---
```

## 三层加载

- **第一层（常驻）**：name + description，几十个技能加起来也就几百 token；
- **第二层（触发加载）**：SKILL.md 正文，建议控制在 300 行以内，只写步骤和约束；
- **第三层（按需读取）**：references/ 下的详细文档和 scripts/ 下的脚本，模型按正文指引决定是否读取或执行。

## 写好 description 是关键

description 不是给人看的，是给模型做路由用的。建议包含三件事：做什么、什么时候触发、触发短语示例。避免"处理 PDF 相关任务"这种无法判定的表述。

# 踩坑点

1. **描述含糊 → 永远不触发**。"处理文档"这种描述模型基本不会匹配，要写明具体动作和触发词。
2. **描述过宽 → 频繁误触发**。两个技能都能匹配"导出"时，模型会随机选一个。技能之间要有清晰边界，宁可在 description 里写明"不适用于 X"。
3. **正文写成百科**。把所有细节塞进 SKILL.md，等于把 system prompt 问题换了个目录存放，违背渐进加载的初衷。细节下沉到 references/。
4. **假设脚本能直接跑**。脚本执行依赖运行环境和权限，跨机器部署前先验证依赖；正文里要写清调用方式和相对路径。
5. **技能不随代码演进**。API 改了技能没改，模型会拿旧指令生成错误调用。把 skills 目录纳入版本管理和 code review。

# 可复用建议

- **一个技能只做一件事**，拒绝"万能技能"。
- **用真实查询做触发测试**：准备 20 条典型用户输入，检查是否命中预期技能、有没有被别的技能抢触发。
- **确定性步骤写成脚本**（格式转换、固定校验），让模型只做判断和编排，不要让它手算。
- **定期审计触发日志**：哪些技能从不触发？要么改描述，要么删掉。不触发的技能不是资产，是噪音。
- **把 description 当路由配置维护**，改完就跑一遍触发测试。

# 总结

Skills 机制的价值不在于"多了一种插件格式"，而在于把上下文当成预算来管理：元数据常驻、正文按需、资源懒加载。实践上，把 description 当路由规则写、把细节从正文里挤出去、用触发日志驱动迭代——这三件事做到位，技能数量涨到几十个，也不会把上下文拖垮。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/4c3c6be2215f7c2d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/0195cab52f42d9db.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/ec7601c1132798f9.png)

