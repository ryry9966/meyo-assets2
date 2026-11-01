---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的一套做法
feedId: 43186
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

Agent 圈这两年一个明显的趋势：能力越挂越多。MCP server、插件、system prompt 里塞满工具说明，动辄几万 token。我自己的 OpenClaw 实例在挂了三个 MCP、十几个自定义指令之后，常驻上下文明显膨胀，长任务里模型经常"忘事"，还出现过明明要查日历却去调网页搜索的情况。

Skills 机制就是针对这个问题的：不把所有能力的完整说明一次性灌进上下文，而是只常驻一份极简索引，模型判断需要时再加载正文。

## 机制怎么工作

一个 skill 就是一个目录，核心是 `SKILL.md`：

```
skills/
  weekly-report/
    SKILL.md
    scripts/gen_chart.py
  log-analysis/
    SKILL.md
```

frontmatter 里写 `name` 和 `description`，正文写触发条件、步骤、约束。启动时 OpenClaw 只扫描 frontmatter，把 name + description 注入上下文，这是**第一层**；当模型判断任务命中某个 description，才去读 `SKILL.md` 正文，这是**第二层**；正文引用的脚本、数据文件再按需读取，是**第三层**。三层递进，常驻成本只有每条 skill 一两行。

```yaml
---
name: weekly-report
description: 当用户要求生成周报、汇总本周提交与任务进展时使用。
  读取 git log 与任务看板，按模板输出。
---
```

## 落地步骤

1. 把现有 system prompt 里"操作指南"类内容拆出来，一段一个 skill 目录，宁小勿大；
2. description 用"什么时候用"来写，而不是"这是什么"。它是检索信号，写错了 skill 就是死的；
3. 确定性操作（画图、转格式、调接口）写成脚本放进 skill 目录，正文只描述何时调用、参数从哪来。模型擅长判断，不擅长稳定执行固定流程；
4. 重启后故意抛几个边界 case 进对话，观察是否命中、是否误触。

## 踩坑点

- **description 写成名词短语**（如"周报工具"）基本不会触发，要写成带触发场景的完整句子；
- **skill 职责重叠会互相抢触发**。"日志分析"和"错误排查"就得在各自 description 里写清边界，否则模型随机挑一个；
- **一个 skill 塞十个流程**，正文太长，第二层加载就失去意义，该拆就拆；
- **脚本依赖要锁死**。我踩过一次坑：skill 里的脚本默认用系统 python，换机器后依赖缺失。现在统一在正文写明解释器路径和依赖来源。

## 可复用建议

- skills 目录进 git，改动走 review，把它当代码而不是配置；
- 盯会话日志里 skill 的触发记录，从不触发或频繁误触就回头改 description——这是调优成本最低的杠杆；
- 和 MCP 分工：MCP 管"接入外部系统的通道"，skill 管"何时、如何做某类事"的知识，两者是互补不是替代。

## 总结

Skills 的价值不在多了一个概念，而在于把上下文当成预算来花：索引常驻、正文按需、脚本兜底。拆完之后我的常驻 prompt 缩了约三分之二，误触率也明显下降。建议从一两个最痛的场景开始拆，别一次性把所有指令搬进去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/890ff3a706acaa06.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/eac43ff4a6cb45bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/fc7f17ee9d7ea6f7.png)

