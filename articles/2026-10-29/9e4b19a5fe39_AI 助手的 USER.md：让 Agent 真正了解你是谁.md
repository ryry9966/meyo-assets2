---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 42873
source: 综合讨论
publishedAt: 2026-10-29
---

## 背景

我们习惯给 Agent 配 AGENTS.md 讲清楚项目怎么跑，接 MCP 给它工具，写一堆插件扩展能力。但很少有人系统回答另一个问题：**Agent 知道你是谁吗？**

现状通常是：每个新会话都重新自我介绍——“我用 macOS，包管理器用 pnpm，回复请用中文，别自动 push”。这些信息散落在聊天历史里，随上下文过期而消失。

## 问题

拆开看是三类：

1. **重复成本**：个人偏好每次手打，或从旧会话里复制粘贴。
2. **偏好漂移**：周一让它先给方案再写代码，周三它又开始直接甩代码，因为没有持久承载位置。
3. **全局与项目混杂**：把个人习惯塞进项目 AGENTS.md，换个项目就丢；或在团队仓库里混入私人配置。

系统提示词是产品方的，AGENTS.md 是项目的，“用户自己”这一层上下文没有归属地。USER.md 补的就是这一层。

## 做法

以 OpenClaw 为例：

1. **定位文件**。工作区根目录的 `USER.md`（默认 `~/.openclaw/workspace/USER.md`），每次会话随系统提示注入。
2. **内容分层**，建议四块：
   - 环境：OS、shell、包管理器、运行时版本
   - 技术偏好：默认语言、框架、测试与格式化工具
   - 行为约束：回复语言、能否自动执行命令、commit 规范
   - 沟通风格：先方案后代码、不要寒暄、不用 emoji
3. **控制篇幅**：60–100 行以内，重要的放前面。
4. **职责分离**：SOUL.md 管人格，AGENTS.md 管操作规程，USER.md 只管“你是谁”，别互相越界。
5. **纳入 git**（dotfiles 仓库），像代码一样 diff 和 review。

一个可以抄的骨架：

```markdown
# 环境
- macOS 15, zsh; Node 22, pnpm
# 偏好
- Python 用 ruff 格式化, 单引号
- 测试: vitest / pytest
# 约束
- 回复用中文
- 未经确认不执行 git push
# 风格
- 先给方案要点, 再给代码
```

## 踩坑点

- **写成愿望清单**。“代码要优雅”“回复要专业”是噪音；“缩进 2 空格、单引号”才是信号。约束要具体到 Agent 可执行。
- **太长**。上下文预算是真实的，300 行 USER.md 会稀释真正要紧的 10 行。
- **放敏感信息**。这个文件每次都进上下文，别放 API key、内网域名、客户名。
- **与 SOUL.md / AGENTS.md 打架**。人格设定写进了 USER.md，或没定义项目规则优先级，表现为“时而遵守时而不遵守”。
- **只写不改**。三个月后它描述的是过去的你，每月扫一遍。

## 可复用建议

- **让 Agent 参与维护**：每隔一两周，让它根据近期对话提议 USER.md 的修改 diff，你只做 review。profile 的自我修正闭环比手写省力得多。
- **用例子代替形容词**：与其说“我喜欢简洁的 commit”，不如贴三条你真实的历史 commit message。
- **当配置不当文档**：改一次提一次 commit，出问题能回滚。

## 总结

USER.md 没有玄学，本质是把“用户侧上下文”从聊天历史里抽出来，变成版本化、可 review 的静态资产。AGENTS.md 告诉 Agent 项目是什么，MCP 告诉它能用什么，USER.md 告诉它对面坐的是谁。三层齐了，行为才稳定得起来。从一个 30 行的文件开始就够。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/985d43b83addca9e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/30ad366167e00ceb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-29/48843e159f5e32d2.png)

