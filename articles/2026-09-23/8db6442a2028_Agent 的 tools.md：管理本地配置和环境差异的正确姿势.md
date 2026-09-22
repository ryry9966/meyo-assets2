---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 38530
source: 综合讨论
publishedAt: 2026-09-23
---

# 背景

跑 Agent 时间长了，几乎都会撞上同一个问题：同一套任务，在这台机器上顺跑，换一台就翻车。笔记本上有 docker，服务器上没有；macOS 路径是 `/Users/xxx`，CI 里是 `/home/runner`；同事机器 node 是 v18，你的是 v22。这些假设一旦写死在提示词里，环境一变就全部失效。

我们在 OpenClaw 的实践里逐步收敛出一个做法：在项目根目录维护一份 `tools.md`，作为 Agent 的“环境说明书”。它不是文档装饰，而是每次任务前会被读取的运行时事实。

# 问题

没有 tools.md 时，环境信息散落在三处：

1. **提示词里写死的假设**——换机器必炸，且很难定位；
2. **Agent 运行时试探**——每次都 `which`、`--version` 摸一遍，浪费 token 和轮次，还可能试错；
3. **散落的脚本和 README**——Agent 不一定读，读了也可能过期。

结果是：Agent 在 A 机器学到的经验，到 B 机器变成误导。

# 做法

我们的 tools.md 大致长这样（节选）：

```markdown
# 本机事实，Agent 先读这里

## 可用
- node v22.11，/opt/homebrew/bin/node
- docker 已装但 daemon 不自启，先跑 `colima start`
- ffmpeg v7，含 libx264

## 禁用/不可用
- 无 snapd，不要用 snap 装东西
- 不要 sudo，需要权限时停下问人

## 环境变量
- 数据库连接读 .env 的 DATABASE_URL，不要猜，更不要写进日志
```

四条要点：

1. **只写事实，不写教程**。每行可验证：名字、版本、路径、状态。
2. **分层**。项目级 tools.md 进 git，机器差异放 `tools.local.md` 进 .gitignore；Agent 按“项目 → 本地”顺序读取，本地覆盖项目。
3. **负面清单和正面清单同等重要**。“不可用”能省掉一整类试错。
4. **可生成的部分用脚本生成**。CI 里放一个校验脚本，把文件里的版本声明和真实环境比对，漂移即报错，防止它变成考古现场。

Agent 侧也加一条约定：运行中发现与 tools.md 冲突，停下来修正文件或提变更，而不是绕过去继续跑。让文件跟着现实走。

# 踩坑点

- **写成散文**。早期我们写过“docker 有点问题，可能要重启”，Agent 不知道怎么执行；改成“daemon 未自启，先跑 colima start”才有效。
- **手抄运行时可探明的东西**。`which` 查得到的路径没必要写，手写的价值在于哪儿都查不到的部分：怪癖、禁用项、入口约定。
- **把密钥写进去**。这份文件会进 git，也会进模型上下文，只写“去哪个文件拿”，不写值本身。
- **让它膨胀**。每次任务都消耗上下文，超过一屏就该删。过期的环境说明比没有更糟。

# 可复用建议

- 新接手一台机器，先花十分钟写 tools.md，再让 Agent 干活，回报很高；
- 模板固定三段：可用 / 禁用与不可用 / 环境变量与入口；
- 把“发现漂移要回写文件”写进 Agent 约定，形成闭环；
- CI 一致性校验是防止文档腐烂最便宜的手段。

# 总结

tools.md 解决的不是配置本身，而是“环境事实如何被 Agent 可靠地知道”。把环境差异从提示词里的暗知识，变成一份带校验、可分层、含负面清单的明文事实，Agent 的跨机器表现会稳定得多。做法不新鲜——本质是给环境做一份 IaC 式的声明——但放进 Agent 工作流里，杠杆比想象中大。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/5e2334be0992305a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/7dd3d401cde10f5e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/28239305f60602be.png)

