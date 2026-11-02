---
title: Markdown 管线：以 Markdown 为中间表示，打通 AI 生成到多平台发布
feedId: 43250
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

Agent 产出的内容大多是 Markdown，这几乎成了默认协议。但"生成完"和"发出去"之间隔着一段不优雅的路：公众号不认 Markdown，只吃内联样式的 HTML；掘金、知乎各有解析器；Hugo/Astro 要 frontmatter；RSS 对转义和绝对链接有要求。我们的做法是把 Markdown 当作整条发布管线的中间表示（IR）：AI 只负责产出标准 Markdown，其余交给代码。

## 问题

早期让模型"为公众号写一版、为博客写一版"，很快暴露三个问题：

1. 同一内容多版本漂移，改一处要同步多处；
2. 格式逻辑散落在 prompt 里，模型偶尔自作主张夹内联 HTML、改标题层级，结果不可复现；
3. 图片链接、代码高亮等平台差异没有统一出口，每次发布都靠手工救火。

## 做法

管线拆成五步，每步是独立可测试的纯函数，以 OpenClaw 插件或 MCP tool 的形式挂给 Agent 调用：

1. **产出规范 Markdown**。prompt 只约束内容，输出后用 JSON Schema 校验 frontmatter（title/slug/tags/date/summary），再跑一遍 markdownlint 风格的规则检查，不合规直接打回重写，而不是人工修。
2. **归一化**。用 unified/remark 解析成 mdast 再序列化：统一换行、去行尾空格、标题层级归一（强制从 H2 起）、链接转绝对路径。这一步保证后续所有转换看到的是同一棵语法树。
3. **图片独立处理**。扫描图片引用，走 upload 接口传到目标平台图床，回写 URL 并生成 image manifest。图片永远不和其他转换混在一起。
4. **平台适配器**。一个平台一个 adapter，输入 mdast，输出目标格式：公众号 adapter 把代码块逐行转成带内联样式的 `<section>`；静态站 adapter 注入 frontmatter 并改写内链；RSS adapter 做转义和绝对化。
5. **快照校验**。每个 adapter 配 golden file，在 mdast 层做 diff；dry-run 先渲染预览，确认后再真正推送。

Agent 在这条管线里的角色只是调度：调用 `normalize`、`upload_images`、`render.wechat` 这些工具，格式转换本身全部是确定性代码。

## 踩坑点

- 公众号会剥掉 `<style>` 块和 class，代码高亮必须逐行内联样式，长代码块会明显膨胀，注意字数上限；
- title 里带冒号不加引号，YAML 解析直接失败，Schema 校验要覆盖转义；
- AI 有时在 Markdown 里夹私货 HTML，html node 要么白名单放行，要么在归一化阶段剥离；
- 表格、脚注在知乎和 RSS 里表现不一致，adapter 里要有降级（表格转列表）；
- 用字符串 diff 对比输出误报多，务必在 AST 层 diff。

## 可复用的建议

- 内容只写一份，Markdown 是唯一事实源，绝不为平台 fork 内容；
- 格式逻辑进代码不进 prompt，prompt 只管内容质量；
- 每个转换保持幂等：同输入必同输出，管线才值得信任；
- 图片与正文分离，先传图回写 URL，再跑格式转换。

## 总结

管线跑通后，发布从"每平台半小时手工活"变成"一条命令加一次预览确认"。真正省时间的不是 AI 写得多快，而是把不可复现的部分从模型手里收回来，交给确定性代码。Markdown 在这里的价值不只是格式，而是让 Agent 和发布系统之间，有了一份双方都能校验的契约。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/26368ec068aebb5c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/3194adc25ea19c9f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/c03eb2169df35966.png)

