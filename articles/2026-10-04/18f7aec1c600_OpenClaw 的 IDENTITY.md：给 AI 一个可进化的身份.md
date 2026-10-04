---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 40427
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

OpenClaw 的 workspace 里有一组 Markdown 文件承担"软配置"：SOUL.md 管行为准则，USER.md 管用户画像，MEMORY.md 管长期记忆，而 IDENTITY.md 回答一个更基础的问题——**这个 agent 是谁**。它不是系统提示词的替代品，而是人格层的单一事实源：名字、性格基调、语气样例、边界。

## 问题

此前我把"人格"散落在三处：网关的 prompt 模板、几段写死的系统提示、以及聊天中"你以后都这样说话"式的口头调教。后果很典型：

- **多入口不一致**：CLI 和微信端像两个不同的助手；
- **行为漂移**：长会话后语气渐变，没人说得清"标准"是什么；
- **调整靠重启**：改一次人格要动部署，且没有变更记录；
- **无法协作**：人格定义散落各处，团队没法 review。

## 做法

**1. 建最小身份文件。** 在 workspace 根目录放 IDENTITY.md，控制在一屏以内：

```markdown
# Identity
name: A-Long
persona: 务实、克制，先给结论再给依据
tone_sample:
- "方案有两个，推荐 B，原因如下。"
boundaries: 不替用户做不可逆决定；不夸大确定性
```

重点是 `tone_sample`：两句真实示例比十句形容词更能约束输出风格。

**2. 单一事实源。** SOUL.md、各网关模板引用它，不要复制内容。改人格只改这一处，避免多处维护导致的不一致。

**3. 让进化受控。** 约定自省机制：用定时任务（或冲突性对话触发）让 agent 输出对 IDENTITY.md 的 unified diff 提案，附触发原因，先落到 memory；人 review 后用 git 合并。"可进化"不等于"自由变异"。

**4. 版本化。** workspace 全量进 git，身份变更的 commit message 记录动机。任何一次人格调整都可回滚、可追溯。

## 踩坑点

- **写太长**：身份文件超过一屏后，约束力反而下降，模型抓不住重点。我砍掉 70% 篇幅后效果更好。
- **职责混杂**：把"每次先检索再回答"这类任务规则写进身份文件会污染人格层。行为规则归 SOUL.md，身份归 IDENTITY.md，别互相渗透。
- **无人审的自改**：让 agent 直接改写自己的身份，几轮之后就会漂移，甚至出现给自己"松绑"边界的倾向。必须人审 diff。
- **多角色共用一份**：两个性格定位不同的 agent 共用一份 IDENTITY.md，结果是互相中和、都不像谁。按角色分开维护。

## 可复用建议

- 身份文件控制在 30 行左右，`tone_sample` 至少给两句；
- 采用"提案—审批"工作流：agent 只产出 diff，永不直接合并；
- 每周跑一次自省任务：对比近七天对话与身份描述的偏差，生成提案；
- 身份变更一律走 PR 流程，把回滚成本降到零。

## 总结

IDENTITY.md 的价值不在于"让 AI 更像人"，而在于把人格变成**可版本化、可 review、可回滚的数据**。当身份像代码一样被对待，"可进化"才是一个工程属性，而不是玄学。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/5055705d613a1da6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/eb561dbc39a64b44.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/df0845f362bb67d7.png)

