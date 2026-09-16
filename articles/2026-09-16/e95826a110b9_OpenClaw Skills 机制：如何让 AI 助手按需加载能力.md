---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37834
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

跑 Agent 时间长了，最直观的感受是上下文越来越贵，也越来越“脏”。早期我把所有指令、工具说明、业务规则全塞进 system prompt，工具列表拉到几十个。结果是 token 成本上去了，模型选错工具的频率也上去了——可选项太多，注意力被稀释。

Skills 机制就是为解决这件事：把能力拆成独立模块，平时只暴露元信息，命中时才加载正文，资源文件按需读取。核心思路是渐进式披露（progressive disclosure）。

## 问题具体是什么

1. 全量注入的 prompt 里九成内容与当前任务无关，模型容易被无关规则带偏；
2. 工具列表过长，函数调用准确率明显下降；
3. 多个项目共享同一助手时，能力无法按项目裁剪，复制 prompt 维护成本高。

## 做法

一个 Skill 就是一个目录，最少只需要一个 SKILL.md：

```text
skills/
  weekly-report/
    SKILL.md          # frontmatter + 指令正文
    scripts/
      fetch_data.py   # 可选：确定性逻辑写成脚本
    references/
      template.md     # 可选：按需读取的资源
```

frontmatter 只有 `name` 和 `description` 两个字段，而 description 是触发判定的唯一依据：

```yaml
---
name: weekly-report
description: 生成团队周报时使用。当用户要求汇总本周工作、整理进度或撰写周报时触发。
---
正文写操作步骤、约束和示例输出……
```

三层加载逻辑：

- 启动时只读入所有 skill 的 name + description（每条几十 token）；
- 用户请求命中描述时，加载该 skill 完整正文；
- 正文里以路径引用的资源（模板、脚本），执行到才读取。

与 MCP 的分工：MCP 提供工具接口（“能调用什么”），Skills 提供操作知识（“该怎么用、什么流程”）。两者叠加，不是二选一。

## 踩坑点

- **description 写不好，skill 等于不存在。** 太模糊（“处理文档”）永远不触发；太宽泛（“帮用户做事”）每次都触发。我的经验是照着用户真实会说的话来写触发条件。
- **别在正文里堆常驻知识。** 每次都要用的规则放回 prompt；做这件事才需要的才进 skill。
- **脚本路径写死绝对路径，换机器就挂。** skill 内引用资源一律用相对路径，交给加载器解析。
- **一个 skill 干多件事会互相污染触发条件。** 拆。
- **改了没生效，先查加载路径。** 通常是缓存或目录指向问题，别急着改内容。

## 可复用建议

- 描述公式：做什么 + 什么时候用，嵌入 2~3 个用户可能的原话短语；
- 单一职责，正文控制在 200 行以内，超了就拆，或把细节挪进 references；
- 确定性步骤（格式转换、数据拉取）写成脚本，让模型只做编排和判断，省 token 且结果稳定；
- 建一个最小触发测试集，每次改 description 跑一遍，避免上线后“隐身”。

## 总结

Skills 的价值不在“多了一种插件”，而在把上下文当成预算来管理：常驻的只留索引，细节按需加载。实践下来，同样的任务 token 消耗降了一截，触发准确率反而更高。建议先从你最常重复口述的那类操作入手，抽出一个 skill 跑两周，再决定铺开。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/b21e7d3e542a9cba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/540b15568ef737d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/f6715f7cdfca3d03.png)

