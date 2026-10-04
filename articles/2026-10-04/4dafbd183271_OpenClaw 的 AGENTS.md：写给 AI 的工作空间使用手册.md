---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 40455
source: 综合讨论
publishedAt: 2026-10-04
---

# 背景

README 写给人看，AGENTS.md 写给 Agent 看。在 OpenClaw 的 workspace 机制里，Agent 每次进入一个工作空间，都会把根目录的 AGENTS.md 作为第一手上下文读入——相当于新同事入职第一天拿到的那份《项目须知》。这个约定已被多家编码 Agent 工具采用，OpenClaw 沿用了它，并把它扩展为 workspace 级的使用手册。

# 问题

没有 AGENTS.md 的 workspace，Agent 只能靠猜：

- 构建用 npm 还是 pnpm？测试命令是什么？
- 哪些目录是生成物不能动？哪些配置碰不得？
- 挂了一堆 MCP 工具，什么时候该用哪个？

结果是每个 session 都要重复交代，换个模型行为就漂移。团队成员各自的 prompt 片段散落在聊天记录里，无法 review，也无法沉淀。

# 做法

我们目前的 AGENTS.md 约 120 行，结构如下：

1. **环境与命令**：运行时版本、包管理器、build/test/lint 的精确命令，每一行都要求能直接复制执行。
2. **目录地图**：一级目录各配一句“是什么、能不能改”。
3. **约定**：错误处理风格、命名规则、提交信息格式。
4. **边界**：明确禁止项——不碰 CI 配置、不改锁文件、不提交 .env。
5. **工具使用**：允许调用的 MCP server 清单，各自的适用场景与参数习惯。
6. **自检**：Agent 完成任务后应运行的验证命令。

子项目可放自己的 AGENTS.md，OpenClaw 按“就近覆盖”规则读取：Agent 进入哪个目录，就叠加哪一层。全局约束放根目录，特例放子目录。

核心原则：把它当 runbook 维护，而不是介绍页。Agent 犯同样的错两次，就补一条规则进去，随 PR 一起 review。

# 踩坑点

- **写成了散文**。“保持代码整洁”这类话 Agent 无法执行，改成“新增函数必须带单测，跑 `pnpm test` 验证”。
- **与 README 重复**。两份文档必然漂移。README 讲业务，AGENTS.md 讲操作，重叠内容互相引用，不要复制。
- **太长**。超过两三百行，靠后的规则基本被忽略。关键约束放最前面。
- **命令过期**。一条错误的构建命令比没有更糟，Agent 会反复重试。我们加了 CI 任务，定期把文件里的命令真实跑一遍。
- **写入敏感信息**。AGENTS.md 会进入模型上下文，任何 token、内网地址都不要出现。
- **子目录文件冲突**。在根文件里写清优先级，层级最多两层，再深就没人维护了。

# 可复用建议

- 每一行要么是可执行命令，要么是可验证的约束，否则删掉。
- 进 git 管理，改动走 PR——它是人与 Agent 之间的接口契约，接口变更应当被 review。
- 指定 owner 和刷新节奏，像对待 oncall 文档一样对待它。
- 新 workspace 从模板起步，宁可先留空占位，也不要写想象出来的命令。

# 总结

AGENTS.md 的价值不在写得多漂亮，而在把口头知识变成 Agent 每次都能读到的确定性输入。成本极低——一个 markdown 文件——却决定了 Agent 是“每次重新猜”还是“按手册干活”。我们的经验是：先写 30 行能跑的，再随踩坑逐步增补，比一次性写全有用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/c06ca588537c78c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/3d709c3a2c81d54d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/7f012f003e6bcaa8.png)

