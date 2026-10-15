---
title: OpenClaw 会话隔离实战：让子 Agent 别把主会话搞脏
feedId: 41634
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

OpenClaw 里，主会话通常承担两件事：维持长期上下文、沉淀记忆。当我们用 spawn 起子 Agent 去跑搜索、批量处理、代码巡检这类子任务时，一个常见副作用是：子任务的过程信息——工具原始输出、中间推理、报错重试——顺着会话通道回灌进主会话。跑几次之后，主会话上下文被撑大，token 消耗上涨，行为开始漂移，甚至记忆文件里混进了子任务的临时结论。

## 问题

实测下来，污染主要有三条路径：

1. **同 key 串台**：spawn 时没指定独立 session key，子 Agent 与主会话共用一条 transcript，历史互相可见。
2. **回传未收敛**：子 Agent 结束时把全部过程摘要回传，一次几千 token。
3. **文件系统共享**：子 Agent 直接读写主 workspace 和记忆文件，把临时状态写进了主记忆。

## 做法

我们收敛成四步，稳定跑了一个多月：

**第一步，强制独立 session key。** spawn 配置里显式声明：

```yaml
spawn:
  isolated: true
  session_key: "sub:researcher:20250611:7f3a"
```

命名规范是 `sub:<agent名>:<日期>:<短id>`，排障时一眼能定位。

**第二步，收窄回传通道。** 工具输出和中间推理留在子会话本地 transcript，只允许通过 `report` 字段回传结构化结果，并设 token 上限（我们用 500）。主会话收到的应该是"结论 + 关键引用"，不是过程。

**第三步，沙箱化文件与工具权限。** 子 Agent 挂独立 workspace，主记忆默认只读；结果写进自己的子目录，由主会话决定是否吸收。工具白名单按任务最小集配置，不给记忆写入类工具。

**第四步，管好生命周期。** 子会话设 idle timeout，结束后 transcript 归档到独立日志目录，供审计，但不进主上下文。

## 踩坑点

- 最坑的一次：某插件内部 spawn 忘了传 `session_key`，子任务历史直接写进主会话，只能手动裁剪 transcript 回滚。现在 spawn 前置校验 key 已固化进模板。
- 子 Agent 默认能读主记忆，"继承"了主会话的语气偏好，输出混入无关上下文。改只读挂载后消失。
- 回传未截断前，一次批量任务 8 个子 Agent 回灌了 6k token。
- 并发子任务曾共写同一个临时 JSON，互相覆盖，主会话拿到空结果。按 key 拆子目录后解决。

## 可复用建议

1. 把 spawn 配置做成模板脚本，key、TTL、回传上限、工具白名单四项必填。
2. 在主会话规则里写明："只消费子 Agent 的 report 字段"。
3. 每周巡检一次子会话归档目录，看有没有异常膨胀的 transcript。
4. 原则一句话：**默认不共享，显式声明共享**。主会话只收结论，子会话消化过程。

## 总结

会话隔离不是加一个开关就完事，它是一组约束：独立的 key、收窄的回传、隔离的文件、有限的生命周期。四者凑齐，子 Agent 才是"用完即走"的工人，而不是往主会话里倒垃圾的室友。这套做法在我们日均几十次 spawn 的负载下稳定运行，主会话上下文体积回落了约五成，供大家参考。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/a7355433045e8129.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/ed2c9d43e05d2823.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/273e65743d2a7731.png)

