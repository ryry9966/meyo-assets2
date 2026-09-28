---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 39356
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

Agent 写完一篇 Markdown 只是开始。同一篇内容往往要进公众号、知乎、掘金、GitHub README 和 RSS，各平台渲染器差异很大。让 agent 针对每个平台重写一遍，既浪费 token，又会引入内容漂移——改了 A 平台忘了 B 平台。我们在 OpenClaw 工作流里的做法是：agent 只产出一份“规范 Markdown”，格式适配交给确定性的管线去做。

## 问题

差异主要来自三类：

1. **语法子集差异**：脚注、任务列表、`>[!NOTE]` 告警块、数学公式，各平台支持参差不齐；
2. **渲染环境差异**：公众号编辑器只认 inline style 的 HTML，直接粘贴代码块丢高亮；知乎会吞掉一部分标签；
3. **资源差异**：相对路径图片、防盗链图床、锚点 slug 生成规则，各平台各一套。

更麻烦的是 AI 生成的 Markdown 常有“过度格式化”倾向：三层嵌套列表、满屏加粗、emoji 标题，这些在部分平台直接排版崩坏。

## 做法

核心原则一句话：**Canonical → Derived，单向转换，源文件永远不改。**

1. **定义规范子集**。写一份 canonical spec：允许的块级元素、标题层级、代码围栏、图片格式，作为 system prompt 的一部分喂给 agent。agent 的输出必须落在子集内，从源头掐掉过度格式化。
2. **AST 化，不碰字符串**。用 remark 解析成 mdast，所有转换在 AST 节点上做。用正则改 Markdown，迟早被嵌套代码块教做人。
3. **平台 profile**。每个目标平台一个 transform 配置，声明“支持什么、降级策略是什么”。比如公众号 profile：代码块 → inline-styled HTML；脚注 → 文末“参考”段落加序号；告警块 → 引用块加前缀符号。
4. **资源归一**。图片统一转存到自有图床，替换为带尺寸的绝对 URL；文内锚点链接按平台规则重写，无法保证的降级为纯文本。
5. **lint 兜底**。转换后跑一次平台能力检查，遇到不支持节点直接报错，而不是静默降级。静默降级是排版事故的头号来源。
6. **接入 agent**。“发布到 X”做成一个 skill：agent 只调 pipeline CLI，参数是 canonical 文件路径加平台名，转换细节对 agent 透明。

伪代码示意：

```js
const ast = remark.parse(canonical);
for (const rule of profiles[platform].rules) {
  ast = rule(ast);
}
lint(ast, profiles[platform].allowed);
```

## 踩坑点

- 公众号粘贴 HTML 会丢 `<style>` 块，且 class 属性被过滤，所有样式必须 inline；
- 掘金/知乎的锚点 slug 算法和 GitHub 不同，中文标题的锚点链接基本必坏，跨平台目录链接建议一律降级；
- YAML frontmatter 转纯 HTML 时容易原样泄进正文，转换前先剥离；
- AI 偶发输出嵌套围栏（code fence 里又有 ``` ），解析器会提前截断，lint 里加一条“围栏标记长度大于 3”的检查；
- 硬换行规范不一（行尾两空格 vs 反斜杠），统一在 AST 阶段转 soft break，再按平台策略展开。

## 可复用建议

- 先维护一张“特性 × 平台”能力矩阵，lint 规则直接从矩阵生成，别手写散落各处的 if/else；
- mdast 解析结果做缓存，一次解析、多平台复用；
- 降级要显式：转换日志列出每个被降级的节点和原因，发布前人工过一眼；
- agent 侧只约束“能写什么”，不约束“怎么转”。职责分离之后，prompt 会短很多，行为也稳定得多。

## 总结

格式适配是工程问题，不是提示词问题。让 agent 产出受限的规范输入，把脏活交给确定性的 AST 转换，链路才可测、可回滚。管线上线后，同一篇稿子五端发布从逐个手排半小时变成一条命令，且内容漂移为零——这才算把 agent 的产出真正接进了发布环节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/f5449e6bcb85e532.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/74cf31aceb5d6f3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/0f0a5fdb5bac6b85.png)

