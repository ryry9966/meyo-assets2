---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 41538
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的主会话（main session）本质是一份持续累积的上下文：用户偏好、项目约定、历史决策都靠它维持。一旦开始派生子 Agent——跑检索、批量改文件、调 MCP 工具——一个老问题就会浮现：**子 Agent 的过程记录，该不该进主会话？**

## 问题：三种典型的"污染"

1. **工具输出直写主会话。** 子 Agent 读了个 3000 行的日志，全文进 transcript，下一轮 context 直接被撑爆。
2. **失败重试留痕。** 子 Agent 内部重试 5 次，5 段报错栈全留在历史里，模型后续推理容易被噪音带偏。
3. **共享状态被改写。** 子 Agent 往共享 memory 或会话变量里写东西，主会话的"记忆"被悄悄替换。

共同点是：把"执行过程"和"执行结果"混为一谈。主会话需要的是结果，不是过程。

## 做法：临时会话 + 结构化回传

以我们目前的用法为例（字段名以你本地版本为准）：

```yaml
subagent:
  session: ephemeral          # 独立 transcript，跑完即弃
  tools: [fs:read, web:search] # 只给必要权限
  return:
    - status
    - summary      # ≤ 500 tokens
    - artifacts    # 产物文件路径，不贴正文
```

1. **子 Agent 用临时会话。** spawn 时声明 ephemeral session，它拥有自己独立的 transcript。
2. **定义回传契约。** 只返回结构化 summary，禁止把 transcript 原文塞进 return。
3. **父会话只消费 summary。** 子会话记录不注入、不入库，只把 summary 和 artifacts 路径写进主会话。
4. **transcript 落盘留档。** 丢弃 ≠ 不存：子会话完整记录写到 `runs/<task-id>/`，排障时翻得回来。

一句话原则：**父 → 子传任务规格，子 → 父只传摘要，单向数据流。**

## 踩坑点

- **让子 Agent"返回完整日志"等于没隔离。** 见过有人把 return 设成 transcript 全文，主会话照样爆炸。
- **session 隔离 ≠ 权限隔离。** 会话独立了，工具权限还得单独收紧，否则子 Agent 依然能写共享资源。
- **共享 memory 要按会话加命名空间。** 否则并发两个子 Agent 会互相覆盖。
- **摘要的摘要会失真。** 多层派生（子再派孙）时，关键事实放在结构化字段里传递，别指望层层 summary 还原现场。

## 可复用建议

- **契约先行：** 每个子 Agent 上线前先定 return schema，status / summary / artifacts 三件套能覆盖 80% 场景。
- **给 summary 设 token 预算，** 超限让子 Agent 自己先压缩，别把压力转嫁给主会话。
- **留档路径规范化，** 出问题按 task-id 回放完整过程，隔离和可追溯不冲突。

## 总结

session 隔离解决的是**上下文卫生**问题，不是安全问题。记住三件事：过程与结果分离、单向数据流、留档可回放。做到这三点，子 Agent 派得再多，主会话也还是干净的那一份。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/bdfc56cc5bf1f096.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/75ccc0d6b26f4926.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/c6efdb00ab3b1e24.png)

