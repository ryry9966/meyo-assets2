---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 42673
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景

OpenClaw 的 Skills 机制解决的是一个老问题：Agent 的上下文窗口有限，但你想给它的能力越来越多。早期做法是把所有工具说明、操作流程全塞进 system prompt，结果是 prompt 膨胀、token 费用上涨，模型在几十个工具里挑花眼。Skills 借鉴渐进式披露（progressive disclosure）的思路做了三层加载：启动时只注入每个技能的 name 和 description；模型判断相关后才读入正文指令；正文引用的脚本和参考文件，再按需读取。

## 问题

我自己的 workspace 里接过五六个 MCP server，加上几条常驻自动化流程，光工具描述就吃掉小几千 token。更麻烦的是选择准确率：低频能力（比如“生成周报格式”）明明常驻在上下文里，模型该用的时候想不起来，不该用的时候又乱触发。

## 做法

1. **建目录。** 个人级技能放 `~/.openclaw/skills/<skill-name>/`，项目级放 workspace 的 skills 目录，启动时会扫描两处。
2. **写 SKILL.md。** frontmatter 里 name 用小写连字符；description 是模型决定是否加载的唯一依据，写“做什么 + 什么时候用”。
3. **正文写操作指令：** 步骤、约束、输出格式，控制在几百 token。
4. **确定性操作写成脚本**放进技能目录，让模型调用脚本而不是自己发挥。
5. **大块参考资料放 references/ 子目录**，正文只留一句“需要细节时读取 xxx”。
6. **重载后用一句自然语言测试触发**，观察日志确认 description 是否命中。

## 踩坑点

- **description 写给人看而不是给模型看。**“处理数据”这种描述，要么永不触发，要么什么都往它身上套。要写出具体动作和场景关键词。
- **技能职责重叠。** 两个技能都认为自己该上，行为就不稳定。一个技能只管一件事。
- **正文太长。** 触发一次吃掉上千 token，懒加载就失去意义了。
- **脚本有副作用**（写文件、发消息）但正文没声明确认要求，容易误操作。
- **技能和 MCP 工具重复覆盖**同一能力，优先级混乱。我的分工原则：MCP 管原子能力，Skill 管流程和方法论。

## 可复用建议

- description 模板：“<动作短语>，当用户<场景关键词>时使用。”
- skills 目录纳入 git 管理，改技能像改代码一样走 commit，回滚有据可查。
- 低频且步骤固化的能力适合做成 skill；高频简单查询交给 MCP 工具。
- 定期清理：三个月没触发过的技能直接下掉，别舍不得。

## 总结

Skills 的本质是用三层结构换上下文效率：描述常驻、正文懒加载、文件按需读。它不增加模型能力，只改变能力的暴露方式。把 description 当成给模型的检索索引来写，把正文当成操作手册来维护，这套机制才能真正省 token、提准确率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/165ce140c2cbefb0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/ef56c4c36fdc86a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/5af76d4dab79abe1.png)

