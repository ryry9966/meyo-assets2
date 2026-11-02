---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 43251
source: 综合讨论
publishedAt: 2026-11-02
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景

AGENTS.md 是一个正在被越来越多 agent 工具采纳的简单约定：在工作空间里放一个纯 Markdown 文件，agent 启动任务时自动读取，作为项目级上下文。OpenClaw 的加载顺序是：全局 `~/.openclaw/AGENTS.md` → 工作空间根目录 → 当前子目录，就近规则覆盖上层。一句话理解：README 是写给人看的，AGENTS.md 是写给 AI 的 onboarding 文档。

## 没有它会怎样

- 每个新会话都要重复交代构建命令、测试入口、目录约定，上下文被寒暄占满。
- agent 只能靠猜：Node 项目默认 `npm install`，但你的仓库锁的是 pnpm；它好心顺手 `git push`，而你只想让它改文档。
- 教训不沉淀：上个会话踩过的坑，下个会话原样再踩一遍。

## 怎么写：四步

1. **最小可用版**。先写六段：项目一句话定位、常用命令（build/test/lint）、目录结构要点、风格硬约束、明确禁区、任务验收标准。
2. **分目录拆分**。monorepo 在 `frontend/`、`backend/` 各放一份，写局部约定，避免根目录文件膨胀。
3. **接上工具链**。写清 MCP 工具的调用偏好（比如优先用内部检索工具）、插件触发条件，以及"改完必须跑 make verify"这类验收命令。
4. **持续迭代**。agent 每犯一次错，补一条可执行规则；定期清一次失效条目。

骨架示例：

```markdown
# 项目：数据同步服务
## 常用命令
- 测试：pnpm test（不要用 npm）
- 提交前必须通过：make verify
## 禁区
- 不要修改 generated/ 下任何文件
- 未经确认不要执行 git push
## 验收
- 涉及 schema 变更时，先跑 scripts/diff-schema.ts 并附输出
```

## 踩坑点

- **写成百科全书**。超过一两百行后关键指令被稀释，agent 反而抓不住重点。手册要短，细节放子目录文件或链接。
- **指令不可执行**。"遵循最佳实践"等于没写，每条规则都要能落到具体命令或边界上。
- **把秘密写进去**。文件全文会进入模型上下文，token、内网地址不要放。
- **双头维护**。与 README 重复的内容迟早改漏，让 AGENTS.md 引用文档，而不是复制内容。
- **忽略覆盖机制**。子目录文件会覆盖上层同名约定，全局规则"失效"时先查这一层。

## 可复用建议

- 用六段式模板起步：定位 / 命令 / 结构 / 风格 / 禁区 / 验收。
- 建立"错一次，记一条"的沉淀流程，把排障经验转成规则。
- 把 AGENTS.md 纳入 code review，像对待代码一样对待它。
- 验证方法很朴素：开一个全新会话跑一个典型任务，看它是否照手册执行。没照做，先怀疑写法，再怀疑模型。

## 总结

AGENTS.md 的价值不在"让 AI 更聪明"，而在把散落在个人脑子里的隐性工程约定显性化、可版本化。成本极低——一个 Markdown 文件——但能切实减少重复沟通和 agent 犯错率。哪怕你只写一句"提交前跑 make verify"，维护成本就已经赚回来了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/11d483e600ca854e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/b023c12c007c812f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/d68f2eed4fe4f65f.png)

