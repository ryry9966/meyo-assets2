---
title: Markdown 管线实践：从 AI 生成到多平台发布的格式适配
feedId: 38731
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

Agent 产出的东西几乎都以 Markdown 为载体：周报、教程、发布稿、CHANGELOG。OpenClaw 社区常见的做法是让 agent 生成一份 canonical Markdown，再由自动化管线分发到公众号、知乎、掘金、X 等平台。实践中很快会发现：**“一份 Markdown 通吃”不成立**。

## 问题

各平台其实是 Markdown 的方言：

- 公众号编辑器不认 Markdown，要 HTML 且样式必须内联；
- 知乎走标签白名单，脚注和部分表格支持不全；
- 掘金支持 GFM，但代码高亮行为和标准渲染器有差异；
- X 没有格式，长文要拆成 thread；
- 图片是重灾区：外链图床有防盗链，公众号必须走素材接口换永久链接。

再叠加模型输出的不稳定性——偶尔夹带 HTML、LaTeX 公式、结尾的“希望对你有帮助”——下游适配就成了管线里最容易烂掉的一层。

## 做法

我的管线分五步，核心思路是：源头收敛、AST 化、按平台降级。

1. **定契约**。明确允许的 Markdown 子集（例如 GFM 减去脚注和数学公式），写进 agent 的 system prompt，同时用 lint 兜底。在源头禁掉一个特性，比在下游兼容它便宜一个量级。
2. **AST 化**。用 unified/remark（Python 侧可用 markdown-it-py）parse 成 AST，所有转换基于节点操作，不用正则。正则对嵌套列表和代码块内的“假语法”没有免疫力。
3. **平台 profile + adapter**。每个平台一个 adapter，声明允许的节点类型和降级规则：宽表格在移动端降级为加粗列表；数学公式渲染成图片或直接剔除；代码块语言别名归一（sh/bash/shell 统一成一个）。
4. **样式与内容分离**。公众号链路是 mdast → HTML → CSS 内联（用 juice 或自渲染），绝不让模型生成带 style 的 HTML——那是不可测试的。
5. **校验与幂等**。发布前 dry-run：检查链接可达、图片尺寸、字数上限、寒暄句；发布后记录 sourceHash → platformPostId 映射，重试不会重复发。

在 OpenClaw 侧，我把每个 adapter 包成独立的 MCP tool，agent 编排时只面向 canonical Markdown，发布动作和格式适配彻底解耦。

## 踩坑点

- **CJK 加粗失效**：`**中文**` 在部分 CommonMark 实现里因 flanking 规则不生效，parse 前要做一层预处理。
- 代码块内容含 ``` 时改用四反引号围栏，否则会被直接截断。
- 用正则删 HTML 注释会误伤代码块里的示例，这类清理应回到 AST 层做。
- 外链图片失效是上线后最常见的静默故障，入库时同步转存自有图床。
- X thread 拆分要落在句子边界，且绝不能把 code block 拆到两段。

## 可复用建议

- adapter 写成纯函数，配 golden file 快照测试，改 profile 时 diff 一目了然。
- canonical Markdown 留档，作为唯一 source of truth；各平台产物都视为可重建的缓存，不要手改。
- 适配器独立成包或独立 MCP server，不要和业务 agent 耦合。

## 总结

多平台分发的复杂度不在“写”，在“适配”。把契约、AST、adapter、校验四层拆开，每层都可独立测试，管线就从玄学变成工程。最大的收益往往不在下游做更多兼容，而在上游少生成不该生成的东西。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/5aa064d0d07ff76b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/7960cccdd2066bdc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/2d3ff6b91c327f33.png)

