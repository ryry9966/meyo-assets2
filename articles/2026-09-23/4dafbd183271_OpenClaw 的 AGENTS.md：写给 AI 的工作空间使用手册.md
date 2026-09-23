---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 38610
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

AGENTS.md 的定位很朴素：README 是写给人的，AGENTS.md 是写给 agent 的。OpenClaw 每次在 workspace 里开工，都会读取这个文件，作为理解当前工作空间的行为入口。它不是配置文件，不参与程序逻辑，只影响 agent 对这个项目的"常识"。

## 没有它会怎样

- 每次会话都要人工重复交代：用 pnpm 不是 npm、测试跑 `make test`、`gen/` 是生成物别碰。
- agent 凭猜测选命令，猜错就浪费一轮工具调用和 token。
- 改动风格漂移：命名、目录结构、错误处理各干各的。
- 危险动作没有边界：随手重写配置、改动 CI 文件。

本质问题是：工作空间的隐含知识只存在你脑子里，每次对话都要重新灌一遍。

## 怎么写

**位置**：workspace 根目录放一份；monorepo 可在子包内再放，离被改文件越近的优先级越高。

**篇幅**：建议 60 行以内，分六块：

1. 项目一句话概述 + 技术栈；
2. 命令清单：安装/构建/测试/lint，直接给可复制执行的命令，不要转述；
3. 目录地图：哪里放什么，哪些是生成物不可手改；
4. 约定：命名、提交信息格式、错误处理方式；
5. 边界：明确"不许动"的路径和需先确认的操作；
6. 验收标准：声称完成前必须跑什么。

示例片段：

```markdown
# AGENTS.md
- 包管理器：pnpm，不要用 npm/yarn
- 测试：pnpm test（任何改动后必须跑）
- src/generated/ 为生成物，禁止手动编辑
- 提交信息：conventional commits
- 改 .github/ 下任何文件前先询问
```

## 踩坑点

1. **写成给人看的散文**。agent 要的是祈使句和具体命令，"代码要优雅"等于没写。
2. **太长**。它每个会话都进上下文，大段解释会稀释关键规则，模型对中段内容的注意力也偏弱。
3. **与事实冲突**。AGENTS.md 说 npm、CI 里是 pnpm，agent 会摇摆。让 Makefile / lint 配置做唯一事实源，AGENTS.md 里只引用。
4. **写完就腐烂**。依赖升级、目录调整后不同步，过时规则比没有更糟。
5. **写入敏感信息**。密钥、内网地址会进上下文，也会进 git。

## 可复用的做法

- **增量维护**：agent 犯一次错，就加一条规则。把它当"事故驱动的 onboarding 文档"。
- **进版本库**：PR 里像 review 代码一样 review 它的变更，避免单人暗改。
- **定期验收**：开个全新会话，只说"加一个 xx 接口"，看它是否自动跑测试、守住约定；不过关就补规则。
- **团队场景**：它顺带成了新人的快速上手文档，一份成本、两份收益。

## 总结

AGENTS.md 的价值在于把"每次对话的口头交代"沉淀为仓库里的持久资产。写它不超过半小时，之后每个会话都在省时间。原则就三条：短、具体、与真实命令一致。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/4eca764b89a1e0de.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/580b5180ba1c2041.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/1f8fd871a62b60dc.png)

