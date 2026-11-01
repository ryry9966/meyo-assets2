---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43140
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

我们的内容流程已经稳定为：Agent 起草 → 人工修订 → 多平台分发。AI 输出的统一是 Markdown，但下游五花八门：静态站要 front-matter，公众号要内联样式的 HTML 加自家图床，知乎、掘金的编辑器各有各的兼容性。早期每个平台手动改一遍，一周三篇、发四个渠道，格式工作比写作本身还费时间。后来把这件事做成了管线，这篇记录核心设计与踩坑。

## 问题拆开是三类

1. AI 输出不稳定：标题层级漂移（有时一级有时三级）、代码块语言标签缺失、偶发把整篇包进一层多余的代码围栏。
2. 平台限制：公众号防盗链导致外站图全裂、外链会被吞；部分平台表格和公式语法不兼容。
3. 状态不可追溯：发布记录缺失，重跑脚本就重复发布，或把线上文章覆盖成草稿。

## 做法

原则一句话：一条规范化链路 + N 个平台适配器；不要用正则改 Markdown，全程走 AST。

**第一步，规范化。** 用 remark（mdast）做 lint/fix：标题压平到单一 H1、补全代码块语言、HTML 节点尽量转回 Markdown、统一中英文间距。规范化后的 canonical Markdown + front-matter 是唯一事实源。

**第二步，适配器。** 每个目标平台一个插件，接口统一：

```js
// 每个适配器实现两个方法
transform(ast) -> 平台格式   // 公众号：内联样式 HTML + 图床重写
validate(ast) -> issues[]   // 发布前静态检查
```

公众号适配器最重的三件事：图片先走上传 API 换成站内 URL 并回写节点；外链降级为文末引用列表；样式全部内联（思路同邮件模板）。静态站适配器最简单，注入 front-matter 落盘即可。

**第三步，校验与发布。** validate 全绿才调平台 API，默认只发草稿、人工点发布；发布成功后把平台文章 ID 和内容 hash 写进本地清单，幂等去重全靠这张表。

## 踩坑点

- 最大坑：用正则处理 Markdown。代码围栏里出现特殊字符就全盘皆输，AI 偶尔用四反引号嵌套围栏更是直接漏匹配。换 remark 后这类 bug 清零。
- 公众号外链图片不处理，隔几天必然裂图。上传加重写必须做成发布前置步骤，别想着事后补。
- front-matter 忘记剥离，正文开头挂着一块 `---`，读者一头雾水。
- 公式语法：`$...$` 与 `\(...\)` 各平台渲染不一，适配器里做统一转换。
- 别信“这次只是小改”，清单里没有的 hash 一律先发草稿。

## 可复用建议

- canonical Markdown 定成 schema，front-matter 交给 JSON Schema 校验。
- 所有适配器支持 dry-run：只产出本地工件不发请求，方便 diff 审查。
- 每个适配器配 golden file 快照测试，渲染器升级跑一遍就知道改了什么。
- 在 OpenClaw 的 Agent 工作流里，把 `publish_markdown(target=...)` 封装成 MCP tool，Agent 起草完可直接调用，人只守最后一道草稿审核。

## 总结

这条管线没有高深技术，本质是“先规范化、再分发、发布状态可追溯”。价值在于把格式问题从每次发布的手工劳动，变成一次性的工程投入。后来新增一个分发渠道，基本就是多写一个适配器的事了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/94a5e208abaafb19.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/0f4f5862d7c1494c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/12748323fc078e53.png)

