---
title: OpenClaw 多模型路由实战：GPT 与本地模型的分工边界
feedId: 39087
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 默认是单模型配置，起步阶段够用。但当自动化链路一长，任务异质性就暴露出来了：规划决策、工具调用、批量摘要、敏感数据处理，这些请求的难度、量级和风险完全不同。全走云端 API，成本和合规压力大；全压本地模型，复杂步骤又开始掉链子。

## 问题

两种极端我们都踩过：

- **全云端**：一次跑批任务一天烧掉几十万 token，费用曲线难看；且 trace 里有用户敏感日志，外发不合规。
- **全本地**：多步规划任务失败率明显上升，重试几次把本地推理队列堵死，整条流水线反而更慢。

所以核心问题不是"哪个模型更强"，而是**哪类请求该交给哪个模型**。路由是资源调度问题，不是选型问题。

## 做法

**第一步：给任务打标分类。** 在 OpenClaw 的插件或 hook 层给请求加 route hint，实践中够用的分三类：

1. 规划/决策——难、低频、允许贵；
2. 工具调用/结构化输出——中等难度，但对 JSON 稳定性要求高；
3. 批量摘要/清洗——量大、容错高，最适合本地。

**第二步：写路由表。** 保持声明式，别在代码里散落 if-else：

```yaml
routing:
  rules:
    - match: { task: planning }
      model: gpt-4o
    - match: { task: tool_call }
      model: local-qwen2.5-32b
      fallback: gpt-4o-mini
    - match: { task: bulk_summarize }
      model: local-qwen2.5-14b
```

**第三步：配置 fallback 链。** 本地优先、云端兜底，但要区分失败类型——超时走 fallback，结构化输出错误应该先本地修复重试一次。

**第四步：用历史 trace 回放验证。** 抽 200 条过去的执行记录，按新路由表重放，对比各规则的成功率和 token 成本，再灰度上线。

## 踩坑点

- **本地模型的 tool calling 兼容性**：function call 格式差异会让 OpenClaw 插件解析失败，表象是"模型变笨"，实际是协议层问题。切路由前先跑一遍 MCP 工具 schema 的冒烟测试。
- **JSON mode 不可靠**：本地小模型爱输出带注释或尾逗号的 JSON。在路由层做统一 repair，别让每个插件自己兜底。
- **上下文长度不对齐**：本地 32k、云端 128k，长 trace 一截断，路由判断的依据就变了。路由决策必须在截断之前完成。
- **fallback 死循环**：本地超时→切云端→云端限流→又切回本地。给每条请求设 route hop 上限，比如最多两跳。
- **成本统计粒度**：按请求数看不出问题，要按 task 类型聚合，才能发现" bulk_summarize 占了 70% 成本"这类真相。

## 可复用建议

- 路由按**执行阶段**划分，而不是按 agent 划分——同一个 agent 内部也有便宜步骤和昂贵步骤。
- 敏感数据默认走本地，脱敏后才能进云端白名单。
- 常驻保留 5%~10% 流量直连强模型作为对照组，定期回归路由规则是否还成立。
- Prompt 写成模型无关的，别绑定某家的 system prompt 习惯，否则路由表一动就全要改。

## 总结

多模型路由不是省钱小技巧，而是把模型能力当作可调度的资源来管理。路径很简单：先分类任务，再配简洁的路由表，最后用 trace 回放验证。一个三条规则能说清楚的路由表，远比复杂的加权打分系统更好维护——路由层越简单，问题定位越快。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/4ee5399bc20d646b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/1f72051108458cb2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/4073acfee7ec7d71.png)

