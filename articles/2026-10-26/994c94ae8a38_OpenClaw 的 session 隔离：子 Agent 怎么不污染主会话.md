---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42681
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

用 OpenClaw 做多 Agent 编排一段时间后，最常见的退化路径不是权限失控，而是**上下文被慢慢撑坏**：主 Agent 派一个子 Agent 去抓网页、解析长文档、跑批量检索，结果子 Agent 的中间推理、原始工具输出、一大段无所谓的内容全部回灌进主会话。几轮下来，主会话历史里一半是噪音，token 成本上去了，主 Agent 的回答质量反而下降——它开始引用子 Agent 三轮之前的某句废话。

## 问题本质

子 Agent 的价值在于“干脏活”，但脏活的**过程产物不应该出现在主会话的上下文里**。污染通常来自四个口子：

1. 子 Agent 启动时继承了主会话历史；
2. 子 Agent 的流式输出被直接写回主会话记录；
3. 共享的 memory/scratchpad 没隔离，子 Agent 往主记忆里写东西；
4. 返回值没有长度和格式约束，一条结果吃掉半个窗口。

## 做法

核心原则一句话：**进窄，出窄**。给子 Agent 的上下文要窄，它交回来的东西要更窄。

以示意配置为例（字段名以你手上的版本为准，思路通用）：

```yaml
subagent:
  session: fresh            # 全新 session，不继承主会话历史
  inherit_prompt: false     # 不复制主 Agent 的 system prompt
  tools: [web_fetch, parser]  # 工具白名单，按任务收敛
  ttl: 30m                  # 用完即弃
  return:
    format: json
    max_tokens: 800         # 硬性截断
    fields: [summary, sources, confidence]
```

落地分四步：

1. **隔离启动**：子 Agent 一律用全新 session，主 Agent 只下发任务描述 + 绝对必要的最小上下文（别把整段对话甩过去）。
2. **约定返回契约**：把返回 schema 写进子 Agent 的系统提示——只允许回结构化摘要，禁止原样搬运工具输出。入口处做 schema 校验，不合格直接打回重试。
3. **生命周期收敛**：子 session 设 TTL 和轮数上限，任务结束即销毁；失败的任务单独标记，不要靠自然过期兜底。
4. **日志分离**：子 session 的全量过程记录写到旁路日志，主会话只留摘要。排查问题时看旁路，不靠污染主会话来“留证据”。

## 踩坑点

- **system prompt 重复注入**：默认继承父提示词的场景下，主 Agent 那份几 KB 的系统提示会被每个子 Agent 复制一份。子 Agent 多了，光这一项就很可观。关掉继承，给子 Agent 一份精简的专属提示。
- **流式回显**：前端把子 Agent 的流式输出渲染进主对话流，看起来直观，实际上等于没隔离。展示层可以流式，持久化层只落摘要。
- **共享 scratchpad**：最隐蔽的一类。主 Agent 和子 Agent 共用同一个 memory key，子 Agent 中间状态把主记忆覆盖了，而且事后很难复现。内存读写按 session 前缀隔离。
- **忘记销毁**：长期运行的编排里，僵尸子 session 会慢慢吃掉配额。定期跑一个 session 清点脚本。
- **返回值超限**：契约写了 max 800 token，但没在通道层截断，子 Agent 一激动返回 3000 字，照样进来。校验要在传输层做，不能只靠提示词自觉。

## 可复用建议

- 每次引入新类型的子 Agent，先跑一个**噪音测试**：故意让它刷屏输出，观察主会话上下文是否增长。不增长才算隔离生效。
- 返回契约当成接口对待：有 schema、有版本、有失败语义，而不是“子 Agent 自由发挥主 Agent 自己挑”。
- 子 Agent 数量超过三个后，把隔离策略抽成统一的 spawn 配置模板，别每个任务手写一份。

## 总结

session 隔离不是什么高级特性，本质是一条纪律：**子 Agent 是函数，不是对话伙伴**。传参进去，拿返回值出来，过程留在函数体里。把“进窄出窄”和硬性截断做进配置层，而不是依赖提示词恳求，主会话才能长期保持干净。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/b40ec09e2f87e91f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/59098291b636d5db.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/4b9896229060a078.png)

