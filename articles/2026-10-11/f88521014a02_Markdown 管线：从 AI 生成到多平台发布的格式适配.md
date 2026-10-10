---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 41145
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

用 OpenClaw 这类 Agent 做内容生产，瓶颈往往不在写作，而在发布。同一篇 Markdown 要发到公众号、知乎、掘金、独立博客，你会发现"Markdown"其实是个方言集合：公众号不吃原生语法、掘金严格过滤 HTML、知乎对表格和代码块的支持各有取舍。我们最终的方案是：AI 只负责产出一份规范源文件，格式适配交给确定性代码。

## 问题

最容易踩的坑是直接让 Agent“顺便转个格式”。LLM 做转换有三个天然缺陷：不可复现（同样输入两次结果不同）、长文容易漏段、遇到嵌套结构（代码块里的表格、列表里的代码）容易破坏语义。格式适配本质是 AST 变换，是确定性工作，不该交给概率模型。

## 做法

管线分四层：

1. **规范层**：canonical Markdown = GFM + YAML frontmatter（title、tags、cover、各平台开关）。Agent 产出先过 lint：未闭合代码栅栏、裸 HTML、不支持的扩展语法直接打回重写。
2. **AST 变换层**：用 unified/remark 生态，禁止正则全文替换。每个变换写成 remark 插件——图片链接重写、footnote 降级、任务列表转普通列表，按目标平台组合启用。
3. **适配器层**：每个平台一个 adapter。公众号 adapter 的核心是 HTML 样式全内联（公众号会剥掉 class）加图片转存后重写 URL；博客 adapter 近乎直通，只注入 TOC 和代码高亮。adapter 封装成 MCP tool 暴露给 Agent，Agent 只做编排调用，不碰变换逻辑。
4. **校验层**：golden file 快照测试比对输出 diff，另提供 dry-run 模式先产出预览链接，人工确认后再真正发布。

## 踩坑点

- **正则处理代码块**：代码内容里出现 `#` 标题或链接语法会被误伤，是最常见的事故来源，一切变换走 AST。
- **公众号防盗链**：外链图片发布后会裂图，必须先下载再上传到素材库，才能重写 URL。
- **frontmatter 泄漏**：部分平台导入不识别 YAML 头，会把 `---` 之间的内容当正文渲染，适配第一步应显式剥离。
- **中英混排软换行**：不同渲染器行为不一，有的拼回一行，有的断行。约定段落内不硬换行，靠 lint 强制。
- **AI 隐性杂质**：偶发的未闭合代码栅栏、解释文字混进代码块。lint 拦不住的语义问题，靠 dry-run 人工预览兜底。

## 可复用建议

- 源文件语法宁严勿宽，lint 失败就打回，别让适配器猜意图。
- adapter 无状态、纯函数化，输入输出明确，方便单测和快照。
- 平台指令用 HTML 注释承载（如 `<!-- more -->`），但部分渲染器会剥注释，关键逻辑别只依赖它。
- Agent 的定位是编排者与修正者：调工具、按 lint 报错修改源文件，不亲自做格式变换。整条管线挂进 OpenClaw 定时任务即可无人值守。

## 总结

跑通之后，新增一个发布平台的成本就是一个 adapter 插件加一组快照用例，半天内能完成。经验一句话：让 AI 做它擅长的生成与修正，让代码做它擅长的确定性格式变换，边界划清楚，管线才稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/63fa63ff6baa0238.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/d4988f9c93caadd9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/5125c24d349e3ab7.png)

