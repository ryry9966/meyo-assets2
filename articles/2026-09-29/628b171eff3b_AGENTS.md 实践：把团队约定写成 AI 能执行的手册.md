---
title: AGENTS.md 实践：把团队约定写成 AI 能执行的手册
feedId: 39453
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

用 OpenClaw 跑自动化任务久了会发现一个规律：Agent 的产出质量，很大程度取决于它对工作空间的"先验知识"。人进场会看 README、问同事、翻 wiki，Agent 进场只有代码本身。AGENTS.md 就是补上这一块的机制——OpenClaw 在会话启动时读取工作空间根目录的 AGENTS.md，作为本次任务的上下文约定。可以理解为：README 写给人看，AGENTS.md 写给 AI 看。

## 没有它会出什么问题

- Agent 用 `npm` 装依赖，而项目统一用 `pnpm`，锁文件立刻冲突；
- 直接改了 `generated/` 目录下的生成物，下次构建被覆盖；
- 每个人（每次会话）都重新"试探"一遍项目约定，行为不一致；
- 明明有现成的测试命令，Agent 自己发明一套跑法，结论不可信。

这些不是模型能力问题，是上下文缺失问题。

## 怎么写

**位置**：根目录放一份全局约定；monorepo 的子包目录可以再放一份，OpenClaw 处理子目录任务时会就近合并，子目录规则优先。控制嵌套层级，两层足够。

**内容**：只写 AI 需要执行的东西。一个够用的骨架：

```markdown
# 工作空间约定
## 常用命令
- 测试：pnpm test --filter <pkg>
- 构建：pnpm build（禁止全量构建，太慢）
## 边界
- packages/generated/ 是生成物，禁止手改
- 迁移脚本一律走 pnpm migrate:new
## 风格
- 提交信息用 conventional commits
- 优先改现有文件，不要随手新建 utils
```

**写法原则**：祈使句、可执行、可验证。"写出高质量代码"这种话等于没写；"用 pnpm，不要用 npm"才是有效指令。

**迭代闭环**：这是最关键的一条——Agent 犯过的错，第二次还会犯，除非你把它写进 AGENTS.md。我的习惯是：同一个错误出现两次，就加一条规则。AGENTS.md 不是一次写完的文档，是团队踩坑记录的沉淀。

## 踩坑点

1. **写成 README**。项目历史、roadmap、贡献指南这些对 Agent 是噪音，浪费上下文窗口还稀释重点。
2. **太长**。建议控制在 100 行以内。规则太多，Agent 会漏读，长尾规则等于没有。
3. **根目录与子目录冲突**。比如根上写"测试用 vitest"，子包里写"测试用 jest"。改命令时全局搜索一遍，别只改一处。
4. **过期信息**。命令换了、目录挪了，文件没跟上，比没有更糟，Agent 会理直气壮地执行错误指令。约定变更的 PR 里顺手带上 AGENTS.md，让它过 review。
5. **塞进密钥或环境变量**。AGENTS.md 是普通文件，会被读进上下文，敏感信息一律放环境配置。

## 可复用建议

- 团队维护一份 AGENTS.md 模板（命令 / 边界 / 风格三段起步），新项目复制后删改，比从零写快得多；
- 把 AGENTS.md 的 diff 当代码 review，谁发现 Agent 踩坑，谁提规则补充；
- 定期做一次"清退"：删掉三个月没起作用的规则，保持文件锋利。

## 总结

AGENTS.md 的价值不在文件本身，而在于它把"隐性的团队约定"变成了 Agent 可执行的显性输入。写它的过程，本质上是在替团队梳理一遍工程规范——最后受益的不只是 AI，还有每一个新进项目的人。从今天的一个命令、一条边界开始写，比追求完美模板有用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/51d7447060284471.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ce99601dd9fd03c4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ad8fdd85a0bbe2e5.png)

