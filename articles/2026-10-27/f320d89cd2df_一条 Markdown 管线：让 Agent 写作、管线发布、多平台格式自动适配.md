---
title: 一条 Markdown 管线：让 Agent 写作、管线发布、多平台格式自动适配
feedId: 42697
source: 综合讨论
publishedAt: 2026-10-27
---

### 背景

AI 产出的内容基本都以 Markdown 为载体，但发布端的情况很碎：公众号只接受内联样式的 HTML，知乎和掘金吃 Markdown 但扩展语法支持不一，静态博客要 front-matter，Notion 是块结构。手工逐平台排版费时且容易走样。社区里比较一致的做法是：把“格式适配”抽成一条自动化管线——Agent 只负责写，管线负责发。

### 问题

实际跑起来会发现三层问题：

1. **AI 产出的 Markdown 本身不规范**：标题层级从 H3 开始、加粗滥用、代码块缺语言标记，偶尔还把 YAML front-matter 混进正文。
2. **平台方言差异**：公众号外链图片可能被过滤、正文超链接不可点；脚注、任务列表、GFM 表格在各平台支持参差。
3. **资源问题**：相对路径图片发出去全挂，必须转存或替换。

### 做法

1. **规范源**。定一份 canonical Markdown（CommonMark + GFM 子集），配一份 markdownlint 规则锁死。AI 生成后先过 lint，再跑一段清洗脚本：归一化标题层级、剥多余 HTML、补代码块语言标记。这一步放在输出后处理里，而不是在 prompt 里求模型“好好写”。
2. **AST 转换，不写正则**。用 remark/unified（或 pandoc）解析成 AST，每个平台一个 adapter。公众号 adapter 的核心是把节点渲染成内联样式的 HTML 片段；博客 adapter 只拼 front-matter；掘金基本等于“清洗后的源文件 + 图片替换”。
3. **图片本地化**。转换前扫描全部图片引用，下载、压缩、按平台上传或替换为可达 URL，映射写进 manifest。
4. **幂等发布**。发布前先查 manifest 里该文章在该平台的记录，有则更新而非新建。manifest 记 platform、post_id、url、时间。
5. **接入 OpenClaw**。管线封装成一个 skill（或 MCP 工具），对话里一句“把这篇发到博客和掘金”即可触发，返回各平台链接。人只审稿，不排版。

### 踩坑点

- 公众号编辑器会丢 class，样式必须内联，部分属性直接被过滤。最稳的办法是生成后用无头浏览器渲染一遍再取 innerHTML。
- 代码高亮别指望平台：在 AST 层用 highlight.js 把代码块展开成带内联色的 span，否则公众号端会糊。
- 公众号正文外链不可点，统一降级为文末“参考链接”列表。
- 正则改 Markdown 看着快，遇到嵌套列表和表格立刻翻车，务必走 AST。
- 幂等没做好，重跑一次管线就在掘金发出两篇一模一样的。

### 可复用建议

- 先定 lint 规则，再写 prompt，让清洗脚本兜底，而不是赌模型听话。
- adapter 保持小：一平台一文件，输入 AST、输出平台载荷，配 golden file 做快照测试，渲染结果变了能立刻发现。
- manifest 是整条管线的状态中枢，先建它，再写发布逻辑。

### 总结

多平台发布的难点不在“转 Markdown”，而在源内容规范化与平台差异隔离。管线化之后职责清晰：Agent 写，管线发，人只看最终效果。下一步可以把每平台的发布成功率、失败重试也挂进 skill，让整条链路可观测——这才是自动化的完整形态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/5c7e8e947ca683fe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/cb637520fb963c8a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/e7df86ebb4fe7520.png)

