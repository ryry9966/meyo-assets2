---
title: AGENTS.md：给 AI 的工作空间说明书，我把它当配置代码维护
feedId: 41359
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

AGENTS.md 正在变成一个事实约定：在工作空间根目录放一份纯 Markdown，Agent 每次启动自动读取。对 OpenClaw 来说，它的定位很清楚——SOUL.md 这类文件管"你是谁"，AGENTS.md 管"这个工作空间怎么干活"。它最大的优点是和代码同仓库、同版本、走同一条 review 流程。

## 问题

在用它之前，我的痛点很典型：每次新会话都要重复"测试用 `pnpm test`，别用 npm"；agent 改代码不守 lint 规范；明明接好了 MCP 工具，它却不知道什么时候该用；团队里每个人调教出来的 agent 行为不一致，经验全散落在聊天记录里，人一走就没了。共同点是：知识存在会话里，而不是存在工作空间里。

## 做法

1. 根目录建 `AGENTS.md`，先写六块：项目概览、目录地图、常用命令、代码与提交规范、MCP/插件使用注意、禁区。
2. 控制篇幅。它每个会话都会进上下文，我压在 150 行以内，多用短句和列表。
3. 命令写成可直接回车的形态，路径一律用相对路径。
4. Monorepo 场景下子目录可以再放一份，就近生效；子包的特殊规则不要堆进根文件。
5. 进 git，走 PR。改 AGENTS.md 和改代码一样需要 review。

一个骨架示例：

```markdown
# 概览
一句话说明这个工作空间做什么。

# 目录地图
- src/    业务逻辑，入口 src/main.ts
- scripts/ 运维脚本，shell 只放这里

# 常用命令
- 测试：pnpm test
- lint 修复：pnpm lint:fix

# 规范
- 提交信息用 conventional commits
- 不新增依赖，除非先在 issue 说明理由

# 工具
- filesystem MCP 只允许访问工作空间内路径

# 禁区
- 不改 .github/ 下的 workflow
- 不执行联网安装命令
```

## 踩坑点

- **写成许愿池**：什么都往里塞，重点被稀释，agent 反而抓不住要领。
- **与系统提示词/工具描述冲突**：同一件事两处说法不一致，agent 行为会很随机。
- **写死环境**：绝对路径、本机版本号都别放；API key 绝对不能进，这文件是要进仓库的。
- **过期信息比没有更糟**：重构完忘了更新，agent 拿着旧地图走路。
- **当强约束用**：AGENTS.md 是软引导。想禁止危险命令，去配 hooks、权限白名单和 pre-commit，别指望一段 Markdown 能拦住它。

## 可复用建议

- **两次法则**：同一个错误 agent 犯第二次，就写进 AGENTS.md，这是最低成本的迭代方式。
- **写祈使句，不写散文**："测试命令：pnpm test" 好过一段"我们项目使用 pnpm 进行测试"。
- MCP 工具的注意事项紧贴工具名写，别集中堆一块。
- 定期修剪：每两周过一遍，删掉已被 hooks 固化的条目。
- 自检方法：新人入职先读它。人看不懂的，agent 也看不懂。

## 总结

AGENTS.md 本质上是把"调教 agent 的经验"从聊天记录搬进版本库。它便宜、可 review、可复用，是我目前维护 OpenClaw 工作空间性价比最高的一件事。把它当配置代码对待：小步修改、及时更新、定期修剪。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/189c2355a7ac00c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/a8ca7aa696c79145.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/02c55b390ab54884.png)

