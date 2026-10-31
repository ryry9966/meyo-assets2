---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43111
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

用 OpenClaw 这类 Agent 做内容生产，Markdown 几乎是默认交付格式：模型输出是 Markdown，MCP 工具读写的是 Markdown，插件的输入输出也大多围绕它。真正的麻烦在最后一公里——同一篇稿子要发公众号、知乎、掘金、GitHub README 或静态博客时，你会意识到“Markdown 只是中间格式”，每个平台都有一套自己的限制。

## 问题

我们最初的流程很朴素：Agent 写完，人工复制粘贴。问题很快暴露：

1. 公众号不认标准 Markdown，只接受内联样式的 HTML，`<style>` 标签直接被剥离；
2. AI 输出不稳定：有时代码块不标语言，有时围栏里又嵌一层围栏，有时夹带裸 HTML；
3. 图片相对路径在目标平台全部失效，各平台图床互不相通；
4. frontmatter、脚注、任务列表这类扩展语法，部分平台会原样渲染出来，观感很尴尬。

## 做法

我们把管线拆成四层，核心原则是：**规范化在代码里做，不靠 prompt**。

**第一步，定义方言契约。** 明确 Agent 允许输出的 Markdown 子集：ATX 标题、带语言标注的围栏代码块、GFM 表格（不超过 5 列）、禁裸 HTML、禁脚注、链接一律绝对地址。把它当成 API contract 写进文档，system prompt 只引用、不复述。

**第二步，lint 与归一化。** 产出后先过校验：markdownlint 自定义规则兜底，再加一个归一化脚本处理已知脏数据——剥 HTML、脚注转行内括号、无语言代码块补默认标注、frontmatter 抽出存为元数据。prompt 会漂移，代码不会。

**第三步，渲染适配层。** 用 remark/unified 串联：`remark-parse` 解析 → 自定义转换插件 → 按目标平台走不同 rehype 分支。公众号分支最麻烦，要遍历 AST 给每个元素注入内联 CSS，代码块预渲染成带高亮的 `span`；掘金、知乎走标准 Markdown 透传，只做图片链接重写；静态博客直接透传。

**第四步，工具化。** 把整条管线包成 `publish(target)` 的 MCP 工具暴露给 Agent。Agent 只负责产出规范 Markdown，调用工具之后是纯确定性流程。图片上传按内容 hash 缓存，失败走重试队列。

## 踩坑点

- 公众号对样式清洗比想象中狠：行内样式里 `position`、部分伪类会被剥掉，主题 CSS 要单独维护并定期实测；
- 表格在手机端必炸，超过 3 列就降级成列表；
- 图片上传是异步且限速的，先落本地队列再提交，别让 Agent 同步干等；
- 代码高亮如果用 class，公众号同样会剥，必须编译成 `span` + inline color；
- frontmatter 忘删，发出去就是一篇以 `---` 开头的文章，我们犯过不止一次。

## 可复用建议

1. 方言越窄越好，文档化并版本化，当成接口维护；
2. 每个平台适配器写成纯函数：规范 Markdown + 配置 → 平台 payload，方便单测和回放；
3. 保留 canonical 源文件，渲染结果与源文件做 diff 记日志，排障时能快速定位是哪一层引入的变形；
4. 图片按 hash 缓存 + 引用计数，多平台复用同一张图只上传一次。

## 总结

这条管线的本质，是把“格式适配”从 Agent 的一次性发挥，变成可测试的确定性代码。Agent 的能力边界是生成，工程的价值在于收敛。管住方言、管住资产、管住平台差异，一份 Markdown 才能真正做到一次生成、处处可发。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/635727802fa17d8d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/1a21930a61fe05ee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/bd633dac3997efe0.png)

