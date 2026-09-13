---
title: OpenClaw 会话隔离实战：让子 Agent 干完活就走，别把主会话搞脏
feedId: 37441
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

跑过多 Agent 流水线的人大多见过同一个现象：主会话越聊越慢、越来越贵，往上翻记录，全是某个子 Agent 拉回的网页正文、MCP 工具的原始 JSON、一整屏报错栈。这些过程数据本该留在子会话里，却被一路带回了主线。

在 OpenClaw 的模型里，主会话（main session）是长期演化的上下文，子 Agent 是为单个子任务临时拉起的执行单元。隔离做得是否彻底，直接决定主会话的上下文质量和成本曲线。

## 问题

隔离缺失或不彻底时，典型症状有三个：

1. **上下文膨胀**：子 Agent 的工具输出被原样注入主会话，几轮下来 token 占用翻倍，主 Agent 反而抓不住重点。
2. **状态串扰**：子 Agent 把中间结论写进共享 memory，后续主会话推理被污染——调研型子 Agent 的一个猜测，可能被主 Agent 当成既定事实引用。
3. **工作区冲突**：并行的多个子 Agent 写同一个 scratch 目录，文件互相覆盖，复盘时分不清哪个产物属于哪次任务。

## 做法

我们在内部流水线里固定了四步：

**第一步：用独立 session scope 拉起子 Agent。** spawn 时声明 ephemeral session，工具调用和推理轨迹全部落在子会话 transcript 里，主会话默认不可见。

**第二步：定义显式返回契约。** 子 Agent 结束时不回流原始输出，只返回结构化摘要：结论、关键证据引用、产物路径、失败原因。主会话拿到的永远是“压缩后的接口”，不是过程数据。

**第三步：内存与文件双隔离。** 每个子 Agent 分配独立命名空间的 scratch 目录（建议带 run_id），memory 写入默认指向子命名空间；要进主 memory，必须走一次显式 promote，由主 Agent 或人工确认。

**第四步：收尾即清理。** 无论成功、失败还是超时，都执行 session 关闭 + scratch 归档/删除，避免孤儿会话堆积。

伪配置示意：

```yaml
subagent:
  session: ephemeral          # 独立会话
  return: summary_schema      # 只回结构化摘要
  memory_scope: task/{run_id}
  workspace: scratch/{run_id}/
  on_exit: close_and_archive
```

## 踩坑点

- **把父会话历史整个塞给子 Agent**。最常见的反模式，隔离形同虚设，成本还会成倍放大。只传任务描述加最小必要上下文。
- **插件默认把工具输出流回主事件总线**。部分 MCP 插件的默认行为是全量广播，记得在子 Agent 的插件配置里关掉，或改投到子会话。
- **返回契约定得太紧或太松**。太紧，主 Agent 拿不到关键证据，又得追问一轮；太松等于没隔离。建议先跑 20 个样本任务，人工核对摘要够不够用，再收紧字段。
- **异常路径漏清理**。超时和报错分支经常绕过 on_exit，跑一周后 scratch 里全是没人认领的半成品。清理逻辑要放在统一的收尾钩子里，而不是散落在成功路径上。

## 可复用建议

一句话原则：**默认拒绝写入，显式申请回流**。子 Agent 对主会话的任何影响——上下文、memory、文件——都应该是显式动作，而不是副作用。落地时拿四条自查：

- 返回是否结构化摘要，而非原始输出；
- memory 是否按命名空间隔离，promote 是否需确认；
- 工作区是否按 run 隔离，互不覆盖；
- 异常与超时路径是否同样触发清理。

## 总结

会话隔离不是什么高深机制，本质是给子 Agent 立规矩：过程留在自己屋里，出门只带结论。把这套四步固化成团队模板后，我们主会话的平均上下文占用降了一半以上，“主 Agent 忘性变大、答非所问”这类问题也基本消失。如果你的流水线最近开始变慢，先去检查子 Agent 的回流路径——八成问题出在那。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/6b86b5abeb18f2ee.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/d6056c8bc3692d0a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/27cbb95f383570e6.png)

