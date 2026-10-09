---
title: OpenClaw Skills 机制：让助手按需加载能力的工程实践
feedId: 41078
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

OpenClaw 的 agent 默认靠系统提示词和 workspace 文件决定行为。能力少的时候，把指令全写进 AGENTS.md 没什么问题；但当你接了十几个自动化任务——周报生成、日志巡检、发票归档——全塞进上下文就开始出问题了。

## 问题：上下文膨胀

- 每轮对话都携带全部指令，token 成本随能力数量线性上涨；
- 指令越多，模型对单条指令的遵循率越低，注意力被稀释；
- 不相关指令互相干扰，典型表现是两个格式要求打架。

Skills 的解法是**渐进式披露**：启动时只把每个 skill 的 `name` + `description` 注入上下文（通常几十 token），正文和附属脚本只在命中时才被读取。上下文从"全量常驻"变成"按需加载"。

## 做法

**1. 建目录。** skills 放在 workspace 的 `skills/` 下，一个能力一个文件夹：

```
skills/
  weekly-report/
    SKILL.md
    scripts/gen_report.py
```

**2. 写 SKILL.md。** 头部 frontmatter 是触发依据，`description` 是唯一入口：

```
---
name: weekly-report
description: 当用户要求生成周报、汇总本周工作项时使用。
  输入为项目名，输出 Markdown 周报。
---
```

正文只写执行步骤：先跑 `scripts/gen_report.py` 拉数据，再按模板整理，最后输出。步骤越短越好。

**3. 验证触发。** 新开一个 session，问一句"帮我出个周报"，观察日志里是否出现读取 SKILL.md 的动作。没触发，九成是 description 写得不对。

**4. 脚本化、幂等。** 能写成可执行脚本的逻辑不要写成自然语言散文——脚本出错可以重跑，散文步骤只能靠模型发挥。

## 踩坑点

- **description 写成功能介绍而非触发条件。**"这是一个周报工具"模型不知道什么时候该用，要写"当用户要求……时使用"。
- **多个 skill 描述重叠会互相抢触发**，保持单一职责，一个 skill 只解决一类事。
- **把几百行参考资料直接塞进 SKILL.md**，等于把膨胀问题换个位置。正文只写"何时去读哪个文件"，资料放附属文件按需读。
- **脚本里写死绝对路径**，换机器就挂，用相对 workspace 的路径。
- **改了 SKILL.md 不重开 session**，测到的还是旧缓存，怀疑机制失灵前先排除这个。

## 可复用建议

- description 模板：**触发场景 + 输入 + 输出**，三句话以内；
- 高频重复任务先做成 skill，低频的先留在 AGENTS.md 里观察一段时间再迁移；
- skills 目录进 git，调 description 和调 prompt 一样，需要版本管理和回滚能力；
- 组合能力不靠写大而全的 skill，靠 agent 自己串联多个小 skill。

## 总结

Skills 的本质不是"装插件"，而是**把上下文当成预算来管理**：元数据常驻，正文按需，脚本兜底。实际落地时，写好 description 比写好正文更关键——它决定了能力会不会被用上。建议先用一两个高频任务跑通触发链路，验证加载行为符合预期后，再逐步迁移存量指令。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/61ccca7fb61be88f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/bfe1e033fcb1058d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/0d8e651be4526ea7.png)

