---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 41473
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

Agent 玩到一定规模，都会遇到同一个问题：能力越来越多，上下文越来越脏。MCP server 挂了七八个，再加上自定义工作流和输出格式约定，system prompt 轻松破万 token。结果是模型在简单对话里也背着全套工具说明书，指令互相打架，工具选择的准确率反而下降。

OpenClaw 的 Skills 机制针对的就是这个问题。核心思路是渐进式披露（progressive disclosure）：技能平时只往上下文注入一行元数据（name + description），正文和配套脚本只有在模型判定相关时才被加载。

## 做法

一个技能就是一个目录，关键文件是 SKILL.md：

```markdown
---
name: release-notes
description: 当用户要求整理某次发布的变更、生成 changelog 时使用
---

# 步骤
1. 读取最近 20 条 commit
2. 按 feat / fix / chore 分组
3. 调用 scripts/format.py 生成最终格式
```

要点有三个：

1. **description 写"何时用"，不是"是什么"。** 模型靠这一行决定要不要加载，触发条件写得越具体，误触发越少。
2. **正文用祈使句写流程**，能脚本化的就脚本化。模型执行 `python scripts/xx.py` 比复述一段自然语言可靠得多。
3. **层级要分清**：workspace skills 放项目相关流程，用户级 skills 放个人偏好，别把团队流程和个人配置混在一起。

## 踩坑点

- **description 太泛或太宽。** 写"辅助开发"基本永远不会触发；写"所有与代码相关的任务"则每次都触发，等于没做按需加载。
- **脚本没加执行权限或没写 shebang。** 模型调用失败后不会报错罢工，而是开始"自由发挥"，产出看似走完了流程实则完全没跑脚本——这是最隐蔽的一类问题。
- **一个技能塞多个流程。** 按需加载的粒度是技能级，混装意味着触发一次、加载一堆。
- **技能正文写太长。** 加载进来的正文同样占 token，两三千字的正文就是另一种形式的上下文污染。

## 可复用建议

- 一个技能只解决一类事，宁可拆成两三个小技能。
- 把校验逻辑写成脚本，让模型跑脚本看结果，而不是靠它"记得"规则。
- 上线后定期看日志审计：长期零触发的技能，要么改写 description，要么删掉。
- 同名或功能重叠的技能先合并，触发时的竞争关系在排障时很难定位。

## 总结

Skills 机制本质上不是插件系统，而是一种上下文管理策略：把"知道有这个能力"和"展开全部细节"拆成两步。当你的 agent 工具列表开始失控时，先把 MCP 工具说明和固定流程迁进 skills，通常一个下午就能看到上下文占用和工具调用准确率的改善。欢迎在社区里贴出你的 SKILL.md 写法，触发条件的措辞很值得互相抄作业。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/b9df97686b712644.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/6c225bbeb1709e58.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/20cb4cc311994497.png)

