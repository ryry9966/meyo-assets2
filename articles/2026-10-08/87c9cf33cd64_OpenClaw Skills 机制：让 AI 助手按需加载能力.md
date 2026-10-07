---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 40862
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景：上下文是最贵的资源

跑 Agent 一段时间后会发现，最稀缺的不是模型能力，而是上下文窗口。OpenClaw 里系统提示词、MCP 工具定义、常驻指令都在抢 token。早期常见的做法是把所有能力说明全量塞进 prompt，十个八个技能还好，几十个之后 prompt 膨胀、注意力被稀释，模型经常在不相关的任务上引用错误的流程。Skills 机制就是为了解决这个问题：借鉴渐进式披露（progressive disclosure）的思路，能力说明按需加载，而不是开局全量注入。

## 问题拆开看有三层

1. **成本**：全量注入时 token 消耗随技能数线性增长；
2. **干扰**：无关指令长期驻留，降低命中质量；
3. **依赖错配**：技能引用了本机没装的二进制或环境变量，模型反复试错浪费轮次。

## Skills 是怎么工作的

一个技能就是一个目录：

```text
skills/
  video-clip/
    SKILL.md          # frontmatter + 正文
    scripts/trim.sh   # 可选的附属脚本
```

`SKILL.md` 的 frontmatter 至少包含 `name` 和 `description`，可用 `requires`（bins/env）声明依赖做门控。运行时分三层加载：

1. **元数据层**：启动时只把每个技能的名字加一句话描述注入系统提示，单个几十 token；
2. **正文层**：模型判断当前任务匹配后，才去读取完整的 `SKILL.md`；
3. **资源层**：正文里引用的脚本、参考文件，用到才打开。

技能的搜索路径包括工作区的 `skills/`、`~/.openclaw/skills` 和内置技能；`requires` 不满足的技能会直接从元数据中隐藏，从源头避免模型去调一个不存在的能力。

## 写一个技能的步骤

1. 建目录，用动词短语命名，一个技能只做一类事；
2. `description` 当成"触发语"来写：放用户会真的说出口的话和任务关键词——这是模型匹配的唯一依据；
3. 正文按 runbook 写：前置条件、步骤、命令、边界情况，控制在几百行内，超了就拆 references；
4. 确定性操作落成脚本，正文只写"何时调用、如何传参"；
5. 用 `openclaw skills list` 确认被识别，再用三到五条真实问法验证触发。

## 踩坑点

- **description 太泛**（比如"处理文件"）会永远不触发，太窄又该用时没触发。它是给模型看的索引，不是给人看的简介；
- **巨型技能**把所有内容塞一个文件，懒加载就失效了，宁拆勿合；
- **触发是概率性的**：读不读正文由模型决定，措辞影响命中率，关键流程要在正文里写自检步骤兜底；
- **忘记 requires 门控**：技能引用了未安装的 ffmpeg，模型要浪费好几轮才发现；
- **和 MCP 重复**：MCP 提供工具接口，技能沉淀"什么时候用、怎么组合"的过程知识，二者互补而非替代。

## 可复用建议

- 把个人技能库放进 git，像维护 runbook 一样小步提交；
- 命名和 description 结构统一：做什么 + 何时用 + 依赖什么；
- 定期拿真实会话日志回归触发率，半年没触发的直接删；
- 上线前后对比 token 消耗，验证按需加载的实际收益。

## 总结

Skills 的本质是把 prompt 工程从"一次性堆料"变成"可检索的知识库"：元数据换命中率，正文换确定性，脚本换稳定性。写技能时记住一句话——description 是写给模型的索引，正文是写给模型的 SOP。索引写得准，SOP 写得短，按需加载才真正成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/402e7386df10d8e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/3681cb36d05c4000.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/e2bbac02a6816e63.png)

