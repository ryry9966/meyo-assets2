---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 39462
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

最近把内容生产交给 Agent：写作 Agent 产出 Markdown，之后要发到公众号、知乎、掘金和自己的博客。跑起来才发现，真正耗时的不是生成，而是格式的“最后一公里”——同一份稿子，每发一个平台都要人工修一遍。

## 问题在哪

各平台对 Markdown 的支持参差不齐：

- 公众号：非白名单账号不支持外链、绝大部分 HTML 被剥掉、代码高亮受限、图片必须转存到微信 CDN；
- 知乎：部分 GFM 扩展（任务列表等）不认，HTML 标签直接消失；
- 掘金 / 静态博客：接近标准 GFM，基本透传；
- 脚注、表格内行内代码、数学公式，各家渲染器行为都不一样。

直接复制粘贴，结果必然是每篇都返工。

## 我的做法

核心思路：**单一事实源 + AST 转换 + 平台适配器**。

1. **生成端做约束（style contract）**。在 Agent 的 system prompt 里明确元素白名单：只用标准 GFM，禁 raw HTML、禁脚注语法、表格单元格内不放行内代码。源头约束比事后清洗便宜得多。
2. **规范化**。用 remark（unified 生态）把 Markdown 解析成 mdast，跑一遍自定义 lint：代码块语言标签归一（shell/console 统一为 bash）、中英文之间补空格、相对图片路径转绝对引用。
3. **平台适配器**。每个目标平台一个 renderer，输入 mdast，输出平台可接受的载荷：公众号把 `[text](url)` 转为正文文本加文末引用列表，图片走转存接口替换 URL，样式走 inline style 主题；知乎、掘金做降级兼容；博客原样输出加 front-matter。
4. **资产幂等**。图片转存前按内容 hash 建索引，重跑管线不重复上传。
5. **发布前校验**。dry-run 渲染成预览，并断言关键节点：外链数量为 0、无裸 HTML 标签、front-matter 已剥离。

工程上我用 OpenClaw 的插件机制包了这条 remark 处理链，发布动作接 MCP 工具。Agent 只负责产出 canonical Markdown，管线作为独立 job 跑，两边解耦。

## 踩坑点

- **千万别用正则做格式转换**。代码块里可能包含任意 Markdown 语法，正则会连围栏内的内容一起改。必须先解析成 AST，在节点层面操作。
- AI 很爱生成 LaTeX 公式，而公众号完全不认。要么在生成端禁掉，要么在适配器里降级成图片，别指望平台容错。
- 硬换行问题：部分编辑器粘贴时把单换行合并成段落，中文排版表现为句子粘连。适配器里统一换行策略。
- 公众号行内代码含特殊字符时，序列化要转义，否则粘贴后样式直接崩。
- front-matter 别漏剥，知乎等平台粘贴时会把 YAML 原样显示出来。

## 可复用建议

- 平台输出一律当作构建产物，永远只改 canonical 源文件；
- 适配器接口收敛为 `render(ast, platform)` 一个函数，新增平台只加一个 renderer；
- 用 golden file 做回归测试：每个平台固定几份样例输入，断言输出稳定；
- inline style 主题与内容分离，主题单独版本化；
- 管线默认支持 `--dry-run`，先看后发。

## 总结

多平台发布的痛点本质是“一份内容、多份方言”。解法不复杂：生成端约束、AST 层转换、平台适配器、幂等资产、dry-run 校验。花一两天把管线搭起来，之后每篇文章的发布成本就从半小时人工降到一条命令，Agent 产出的内容才能真正“一次生成、处处可发”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/941721e59003beca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b2aaf9647b3bc4a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c92b23c4a531e256.png)

