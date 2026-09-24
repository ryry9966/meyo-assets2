---
title: 把格式适配从模型手里拿回来：一条 Markdown 多平台发布管线
feedId: 38755
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

AI 生成的默认产物是 Markdown，但发布端各不相同：博客和 GitHub 吃标准 GFM，公众号要内联样式的 HTML，知乎编辑器对表格和代码块支持有限。想让 Agent“一键生成并发布”，卡住你的通常不是内容，而是格式适配。

## 问题

最常见的做法是在 prompt 里直接写“输出适合公众号的 HTML”。我们踩过这个坑：模型对平台规则的记忆不可靠（比如公众号会剥掉 class 和 `<style>` 标签）；同一篇内容两次转换结果不一致；出了问题也没法定位——分不清是生成错了还是转换错了。

## 做法

核心原则一句话：**LLM 只产出语义化 Markdown，格式适配全部交给确定性代码。**

1. **单一事实源**：Agent 产出一篇带 frontmatter 的标准 Markdown（含 title/tags/platforms 字段），这是管线唯一输入，下游禁止改动内容。
2. **规范化**：用 remark-parse 解析成 mdast，做确定性清理——标题统一从 h2 起、补全代码块语言标注、剥掉模型包裹 frontmatter 的多余围栏、图片补 alt。输出仍是 Markdown，但已经干净。
3. **平台适配器**：每个平台一个独立模块，输入 mdast，输出平台格式。公众号走 remark-rehype 后内联样式（只认 style 属性）；博客/GitHub 直接 remark-stringify；知乎把 GFM 表格降级成列表。
4. **资源处理**：图片先本地化、按内容哈希重命名、经平台接口上传后回填链接；manifest 记录 hash→url 映射，保证重跑幂等。
5. **发布与预览**：dry-run 模式先把各平台渲染结果落盘供人工过目，确认后由 OpenClaw 的 MCP tool 触发发布。所有转换器以 MCP tool 形式暴露，Agent 只负责编排。

```js
// 适配器统一接口
async function adapt(mdast, { manifest, dryRun }) {
  return { format: 'html-inline', payload, warnings: [] };
}
```

## 踩坑点

- **别用正则改 Markdown**。代码块里的井号和反引号会把解析彻底带偏，AST 是唯一可靠路径。
- **公众号除了剥 class**，对 `<details>`、外链的处理也和非标 HTML 不同，适配器要按“白名单标签 + 内联样式”写，别信转换库的默认行为。
- **frontmatter 被模型包进 yaml 代码围栏**是高频失误，解析前先判断首行是否为 `---`。
- **幂等**：没有 manifest 时重跑管线会重复上传素材，公众号素材接口有配额限制。
- **可追溯**：给每次 MCP tool 调用带 trace id、中间产物落盘，否则“发布出来的格式不对”根本无从回溯是哪一步改坏的。

## 可复用建议

- 内容与呈现分离是底线。“让模型直接输出平台格式”的捷径，维护成本会在第二次使用时连本带利还回来。
- 适配器保持小而蠢：只封装一个平台的已知规则，平台改版时只动一处。
- lint 当门禁：发布前校验图片 alt、裸 HTML、死链，比事后删帖便宜得多。
- dry-run 产物保留几天，排查格式问题基本靠它。

## 总结

这套管线没有黑科技：remark 生态 + 平台适配器 + manifest 幂等，加起来几百行代码。真正值得守住的是那条边界——**生成归模型，适配归代码，编排归 Agent**。格式适配这件事，确定性永远比聪明值钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/fffacd658d9c8e37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/b7c3bdcebfb498ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/f4c7e806244b843b.png)

