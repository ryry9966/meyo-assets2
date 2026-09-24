---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 38871
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景：为什么能力不能全塞进 system prompt

做 Agent 的人基本都经历过这个阶段：工具越接越多，system prompt 越写越长，结果模型开始漏指令、误调用，token 成本也跟着涨。OpenClaw 的 Skills 机制本质上是对这个问题的工程回应：**能力按需加载（progressive disclosure）**，而不是开机全量注入。

## 问题：上下文是稀缺资源

一次会话里，模型能“稳定注意到”的指令量是有限的。把几十个能力的完整说明全部塞进上下文，等于让模型在几十份文档里自己找答案，注意力被稀释、误触发率上升。我们当时的痛点很具体：助手装了十几个能力之后，明明该走 A 流程却套了 B 的步骤。

## 做法：三个层级拆开写

OpenClaw 的 skill 就是一个目录，核心是一个 `SKILL.md`：

```text
workspace/skills/pdf-report/
├── SKILL.md          # 主文档
├── reference/        # 重参考资料，按需读取
│   └── api-notes.md
└── scripts/
    └── render.py     # 确定性步骤写成脚本
```

加载分三层：

1. **启动时**：只有 frontmatter 里的 `name` + `description` 进入上下文，几十个 skill 也就几百 token；
2. **触发时**：模型判断当前任务命中某个 description，才读入 `SKILL.md` 正文；
3. **深入时**：正文索引到的 `reference/` 文件再按需读取，脚本由模型调用执行。

实操步骤：

- 新建目录和 `SKILL.md`，`name` 用小写连字符；`description` 写清楚“什么时候用、输入是什么、产出是什么”；
- 正文用“适用场景 / 步骤 / 示例 / 注意事项”四段结构；
- 可枚举、确定性的步骤（调 API、转格式之类）抽成脚本，文档只写“什么时候调、参数怎么传”；
- 超过一屏的细节拆进 `reference/`，正文只留索引；
- 改完重启会话（或重载 workspace），直接问助手一句验证是否触发。

## 踩坑点

- **description 含糊 = 永不触发**。写“处理文档相关任务”基本没用，要写成“当用户要求把 markdown 转成 PDF 报告时使用”。description 是唯一的触发依据。
- **把全文塞进 SKILL.md**，按需加载就失去意义，退回上下文膨胀的老路。
- **脚本没写清依赖**（Python 版本、工作目录、环境变量），模型执行失败一两次后可能干脆放弃这个 skill。
- **多个 skill 描述互相覆盖**会误触发，保持“一个 skill 一个职责”，描述里出现重叠关键词时要刻意区分。
- frontmatter YAML 格式错了（缩进、冒号），整个 skill 会被静默忽略，日志未必报错。怀疑没生效时先查格式。

## 可复用建议

- 用 Git 管理 skills 目录，改动可回溯，团队可共享；
- description 固定模板：`当需要<任务>时使用；输入<什么>；产出<什么>`；
- 判断逻辑写成文档给人看，确定性操作写成脚本给模型跑；
- 定期 review：从未被触发的 skill，要么重写 description，要么删掉；
- ClawHub 上的社区 skill 可以先装来读源码，看别人怎么拆层级，再写自己的。

## 总结

Skills 机制没有黑魔法，它把一个朴素原则工程化了：**上下文是预算，能力清单要便宜，能力细节要贵到“用的时候才付”**。把 description 当作触发接口认真写，把确定性步骤下沉为脚本，按需加载才能真正跑起来。对我们的体感变化是：误调用明显减少，长会话后期的指令遵循也稳了很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ad6f54105d786edc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/a933a5eca33b10e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/87a95043ed3486ea.png)

