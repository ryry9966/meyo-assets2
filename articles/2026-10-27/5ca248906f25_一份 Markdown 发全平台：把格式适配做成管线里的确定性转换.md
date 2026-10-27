---
title: 一份 Markdown 发全平台：把格式适配做成管线里的确定性转换
feedId: 42745
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

Agent 产稿天然是 Markdown，但发布端是碎片化的：公众号只接受内联样式的 HTML，知乎编辑器对表格和代码块支持有限，静态博客要 frontmatter，掘金和 Notion 又各有各的方言。手动转一次两次还行，当成日常流程跑，纯靠人肉既慢又容易漏。

我们在 OpenClaw 的内容工作流里跑了一条 Markdown 管线，把"生成"和"发布"之间所有的格式问题收进一个确定性阶段。本文记录设计与踩坑。

## 问题在哪

拆开看是三类：

1. **语法方言**：GFM 表格、footnote、数学公式、mermaid，各平台支持度参差；
2. **样式与安全策略**：公众号会剥掉 class、禁外链、禁 script，等于全部样式得内联；
3. **资源与元数据**：图片路径、frontmatter 字段、摘要要求，各家不同。

另一个隐性问题是 agent 本身：同一 prompt 生成的 MD，heading 层级、列表嵌套、代码块写法（围栏 vs 缩进）都不稳定。不做规范化直接喂下游，adapter 里会堆满 if。

## 做法：一份规范源 + N 个适配器

**Step 1 规范化。** 生成稿先过一遍 canonical pass（用 unified/remark）：heading 层级收敛到 h2 起、缩进代码块统一转围栏、HTML 块转纯 MD、frontmatter 按 schema 校验补全。产出 `canonical.md`，此后只读不改。

**Step 2 适配器层。** 每个目标平台一个纯函数 adapter，输入 AST，输出目标格式：

```text
pipeline/
  canonical.md      # 规范源，只读
  adapters/
    wechat.ts       # remark-rehype + 样式内联，外链转文末参考编号
    blog.ts         # 保留 MD + frontmatter
    zhihu.ts        # 剥方言，表格可选降级为列表
  lint/
    wechat.json     # 禁用语法清单
  assets/           # 图床缓存，按内容 hash 命名
```

图片统一前置处理：检测相对路径和远程图，先传图床/OSS，按 hash 去重缓存，回写 URL 后再进各 adapter。

**Step 3 lint 与 dry-run。** 每个平台一份禁用语法清单，渲染前 lint；渲染后用无头浏览器截图抽查代码块和表格——这两处翻车率最高。

**Step 4 封装。** 整条管线包成 CLI，再注册成 MCP tool（入参：canonical.md 路径 + 目标平台列表）。Agent 在工作流里就能自己完成"生成 → 规范化 → 发布"闭环，人只在 dry-run 结果上点头。

## 踩坑记录

- 公众号对 SVG 支持不稳，mermaid 别指望平台渲染，预渲染成 PNG 插进去最稳；
- footnote 语法在国内多数平台不渲染，要在 adapter 里提前转成"编号 + 文末链接"；
- 不要把平台 HTML 反向转回 MD 二次编辑，round-trip 会丢结构，规范源永远是唯一事实；
- 图片上传有频率限制，hash 缓存 + 失败重试队列，避免重复发布时反复传同一张图；
- 嵌套列表里的代码块，部分渲染器会直接吞掉，规范化阶段限制嵌套深度比事后修补省事。

## 可复用建议

1. canonical.md 不可变，adapter 全部纯函数，出问题能定位到单层，回滚只回滚一层；
2. 建一份覆盖你所有在用语法的 smoke-test.md，改管线先跑它再上线；
3. adapter 尽量写成 remark plugin，社区现成的转换器（rehype-inline、mdast-util-\* 系列）直接复用；
4. 发布动作做成幂等：内容 hash 相同的重复发布直接跳过。

## 总结

核心思路一句话：把格式适配从"发布时的人工劳动"，前移成管线里一层层确定性转换。规范源只此一份，平台差异全部关进 adapter 和 lint 清单。Agent 负责产稿，管线负责让它说得清、发得出——这也是这类自动化该有的边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/4e69a3061458cfc6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/5749c9d10a8f6526.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/40aa1bb7ae729dfc.png)

