---
title: OpenClaw 多模型路由实战：GPT 还是本地模型，别靠感觉拍板
feedId: 39142
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

OpenClaw 里跑的 agent 任务差异很大：有意图分类、字段抽取这类"轻活"，也有多步工具调用、代码生成这种"重活"。我们早期全走 GPT，账单和 P95 延迟都难看；后来试过全切本地，复杂任务成功率明显下滑。落到多模型路由后，核心问题就一个：**什么流量给云端 GPT，什么留在本地**。

## 问题

路由不能拍脑袋，三个约束互相拉扯：

- **质量**：多步规划、复杂 function calling，本地 7B/14B 模型明显吃力；
- **成本与延迟**：分类、格式化、摘要这类高频轻任务，上云又贵又慢；
- **隐私**：涉及日志、内部文档的任务，默认不该出内网。

## 做法

我们在 OpenClaw 的 router 中间件里做了三层策略：

1. **按任务类型打标**。每个 skill/plugin 注册时声明 `task_type`：`classify` / `extract` / `tool_call` / `codegen` / `long_reason`。
2. **粗粒度映射**。classify、extract、格式化输出 → 本地 Qwen 量化版（vLLM 部署）；tool_call、codegen、long_reason → GPT。不要基于 prompt 关键词做路由，规则越细越难维护。
3. **级联兜底**。本地输出 JSON 解析失败、置信度低于阈值、或超时 8 秒，自动 fallback 到云端，并把这次降级写进路由日志。

日志记四个字段：`route`（local / cloud / fallback）、`model`、`latency`、`retry_count`。每周拉一次，某类任务的降级率高于 15%，说明当初分级分错了，直接改映射，不用犹豫。

## 踩坑点

- **本地模型 function calling 不稳**：参数漏字段、类型乱飘。最后靠约束解码加一次 schema 校验重试解决，别指望在 prompt 里写"请输出合法 JSON"。
- **长上下文打爆显存**：路由前先数 token，超过本地窗口 70% 一律上云。
- **隐私判断别交给模型**：让模型自己分类敏感数据，误判一次就是事故。我们改为默认本地，维护一份允许上云的数据源白名单。
- **评测集别太小**：最初用 20 条样例调规则，严重过拟合，换批数据全崩。现在固定 300 条，覆盖全部 task_type。

## 可复用建议

- 起步两层就够：一个本地小模型 + 一个云端旗舰，别一上来做五级级联。
- **默认本地、例外上云**，比默认上云再省钱降级更安全，尤其带内网数据的场景。
- 相同输入的任务加一层缓存，命中率往往比预期高，还直接省掉一次路由决策。
- 新映射先灰度 10% 流量，看一周日志再全量。

## 总结

多模型路由本质是工程权衡，不是模型评测。先用最粗的两层映射跑起来，靠路由日志和固定评测集迭代，比一开始追求"最优路由"务实得多。OpenClaw 的中间件机制足够支撑这套做法，关键是把降级路径和可观测性先建好——路由规则可以慢慢调，出了问题能兜住、能回溯，才是底线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/6c970fbbb1a9afb6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/39a6760628bb47c0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/5edfd6995d273b7d.png)

