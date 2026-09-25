---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 39032
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

Agent 产出内容的默认格式是 Markdown，但发布端五花八门：微信公众号只吃内联样式的 HTML，知乎对表格和代码块支持有限，静态博客反而希望纯净 Markdown + frontmatter。同一段内容手工搬运，格式返工的时间往往超过写作本身。

## 问题

把 AI 输出直接投递到各平台，常见三类故障：

1. **源头脏**：LLM 输出的标题层级经常从二级标题开始、列表缩进 2/4 空格混用、偶尔夹带裸 HTML 或零宽字符；
2. **平台差异**：微信会剥离 class 和 `<style>`，代码高亮直接丢失；表格、脚注、任务列表在部分平台静默失效；
3. **流程不可重复**：靠编辑器插件手工点一遍，Agent 无法接管，也无法做回归测试。

## 做法

核心思路：**单一事实源（canonical Markdown）+ 平台适配器（adapter per platform）**。

**第一步：规范化。** 入口先过 lint/预处理，不依赖模型自觉：

- 标题层级重排，统一从 h2 起步；
- 列表缩进归一、代码围栏统一为三反引号；
- 剥离或转义裸 HTML，替换智能引号，清理零宽字符。

**第二步：中间表示。** 用 markdown-it 或 remark 解析成 AST，后续所有转换基于 AST，而不是对原文做正则替换。

**第三步：平台适配器。** 每个平台实现一个 renderer：

- 微信：AST → 内联样式 HTML，所有样式写进 style 属性，代码块逐行 span 上色；
- 博客：AST → Markdown + frontmatter，顺带产出 slug 和摘要；
- 不支持表格的平台：降级为有序列表，或整体渲染成图片。

**第四步：接进 Agent。** 把 normalize → render → publish 包成一个 MCP tool，参数带 `platform` 和 `dry_run`。Agent 生成完内容直接调用，先跑 dry_run 校验，再真实发布。

## 踩坑点

- 微信对 style 属性也有白名单，个别属性会被静默剥离，上线前用真机预览，别只看本地渲染结果；
- 代码块逐行 span 化后，用户复制代码会出现换行差异，行尾需要额外处理；
- 数学公式各平台渲染器互不兼容，稳妥做法是统一转 SVG 或图片；
- golden sample 要用真实平台的返回结果，本地渲染通过不代表平台接受。

## 可复用建议

- **适配器注册表**：新增平台只需实现 `render(ast)` 接口，主干不动；
- **快照测试**：每个平台维护一份 golden sample，平台改版时跑一遍就知道哪里断了；
- **lint 规则反向注入**：同一份规范沉淀成配置后，可以注入生成端的 system prompt，从源头减少脏输出；
- **版本化**：模板和样式进 git，发布记录存下 AST hash，方便追溯是哪次改动导致样式崩了。

## 总结

这套管线没有黑科技，本质是把格式适配从一次性手工劳动，变成有中间表示、可测试、可被 Agent 调用的工程环节。顺序很重要：先做规范化和 AST，再做适配器，最后才接自动化——反着来，返工会很痛。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/9eeff2836a4ac240.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/6f497d0a4e810523.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/383530207a96fcab.png)

