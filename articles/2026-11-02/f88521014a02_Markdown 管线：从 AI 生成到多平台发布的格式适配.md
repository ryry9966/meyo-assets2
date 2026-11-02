---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43279
source: 综合讨论
publishedAt: 2026-11-02
---

## 背景

Agent 的产出十有八九是 Markdown——它是模型的默认输出格式，也是人机之间的最大公约数。但一旦要真正发出去：博客、公众号、知乎、掘金、Newsletter，Markdown 就不再是终点，而只是中间表示。我在 OpenClaw 上跑了一条自动化发布流，让 agent 定期产出技术内容并分发到三个渠道，一个多月下来，踩的坑几乎都集中在“格式适配”这一层。

## 问题：AI 输出 ≠ 可发布 Markdown

直接把模型输出交给发布器，会撞上三类问题：

1. **模型噪声**：整段回答被外层代码围栏包裹、标题层级跳跃（## 后直接 ####）、列表嵌套过深、代码块语言标签缺失。
2. **平台差异**：公众号不支持锚点、表格在移动端溢出；静态站需要 front-matter，正文平台反而会被 `---` 干扰；任务列表、脚注的支持参差不齐。
3. **资产链接**：图片外链有防盗链，发布后裂图只是时间问题。

## 做法：把 Markdown 当 IR 处理

思路借鉴编译器：先规范化成标准中间格式，再为每个目标写适配器。unified/remark 生态最顺手：

```
raw → normalize → canonical AST → adapter(target) → publish
```

**第一步：收敛。** 剥掉外层围栏，内联 HTML 回转成 Markdown（turndown 思路），用 remark-lint 修标题层级：全篇强制单个 h1、禁层级跳变。这一步只管“干净”，不管“平台”。

**第二步：定版。** 对 canonical AST 注入元数据：slug、tags、summary、cover。静态站目标在这里生成 YAML front-matter，注意 title 里的冒号和引号要转义。

**第三步：按目标适配。** 每个目标是一个纯函数 `(ast, target) -> string`：Hugo/Hexo 注入 front-matter；公众号先转 HTML 再把样式内联到每个标签（可参考 doocs/md 的做法），长表格降级为列表；知乎/掘金走 API 直传 Markdown，但图片要重写成平台图床链接。

**第四步：校验。** 每个目标维护一份 golden file 快照，格式变更 diff 一眼可见；发布前跑链接检查。

按 MCP 的思路，这套东西天然适合封装成一个工具：`adapt_markdown(content, target)`，agent 拿到草稿后按渠道调用，人只审最后一步。

## 踩坑点

- **永远用 AST，别用正则。** 嵌套代码围栏正则处理不了，mdast 能救你。
- **图片重写放最后。** 图一旦传到平台图床，URL 就和内容耦合，中途改稿重传成本很高。
- **别在内联样式之后跑格式化。** prettier 会弄乱精心排的 inline style，格式化只允许发生在 AST 阶段。
- **中英混排粗体**：部分平台渲染中文粗体两侧空格表现不一，宁可在 AST 层自己加空格，不依赖渲染端。
- **`---` 歧义**：正文水平线和 front-matter 分隔符，靠“文档开头第一个”区分，转换前先剥离。

## 可复用建议

1. 只维护一份 canonical Markdown，所有平台是它的视图，改稿只改源头。
2. 适配器写成无副作用纯函数，输入 AST 输出字符串，方便单测。
3. 把 remark-lint 规则集当“合同”，模型输出不合格直接打回重生成，比人肉修快得多。
4. 快照测试 + 链接检查放进 CI，管线坏了在本地就能发现。

## 总结

多平台发布的工作量不在“发”，在“适配”。把 Markdown 当编译器的 IR 对待——规范化、定版、按目标转译、校验——整条管线就从“每次手工修格式”变成“改一个转换函数”。AI 生成解决内容从零到一，格式管线解决从一到 N，两者分开治理，自动化才站得住。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/fdf163ed941cb512.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/342357b1ddca3708.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/3ab00007e09331a5.png)

