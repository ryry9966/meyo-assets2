---
title: Agent 的 TOOLS.md：管理本地配置与环境差异的正确姿势
feedId: 42433
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

OpenClaw 的 agent 每次会话都会加载 workspace 里的 Markdown 文件，其中 TOOLS.md 的定位很明确：给 agent 看的、关于"这台机器上工具怎么用"的备忘。听起来简单，但我见过的大多数 TOOLS.md 要么空着，要么写成一段自相矛盾的散文。这篇帖子把我用下来比较稳的做法整理一下。

## 问题

Agent 对运行环境没有任何先验知识，它的默认策略是试错：

- 你机器上用 pnpm，它跑 `npm install`，多出一个 package-lock.json；
- macOS 上是 `python3`，容器里是 `python`，它猜错就报错重试；
- 本地 Docker Compose 是 v2 语法，VPS 上还是老版本；
- 更糟的情况：它把本机绝对路径写进脚本，换个环境直接崩。

这些不是能力问题，是信息问题。Agent 每次重新探索环境，烧的是 token 和你的耐心。

## 做法

1. **只写事实，不写愿望**。TOOLS.md 里放"能跑通的命令"，不是理想流程。例：`构建：pnpm build（不要用 npm）`。
2. **按机器分节**。workspace 会在多台机器间同步（笔记本 + VPS）时，用明确的 host 分节，各写各自的差异：路径、Python 命令名、Compose 版本。
3. **写禁止项**。哪些目录不要动、不要全局装包、不要提交 .env——agent 对"别做什么"的遵守度，往往高于对长篇解释的遵守度。
4. **高频流程给完整命令**。起服务、跑测试、看日志，各给一条可直接复制执行的命令，agent 不需要自己拼。
5. **敏感内容不进这里**。密钥、内网地址写进环境文件，TOOLS.md 只指路："secret 位置在 ~/.config/xxx，读取方式见 deploy 脚本"。这个文件会整份进上下文，当成公开文档对待。

## 踩坑点

- **写成长文**。超过一屏的 TOOLS.md，在上下文里就是噪音。命令化、列表化，每条一行。
- **过期比缺失更糟**。升级了工具链没更新文档，agent 会按旧信息自信地执行。我加了个 `verify.sh`，列几条核心命令的自检，隔段时间跑一遍，过期项一眼可见。
- **和 README 重复**。两处必然漂移。原则：README 给人看，TOOLS.md 给 agent 看执行事实。
- **把会话日志当文档**。有人直接把上一次 agent 对话贴进去，信息密度极低，清掉重写。

## 可复用建议

模板五段：`Toolchain / Commands / Per-host differences / Do-not / Verify`。多实例或团队场景下，把工具链变更放进 PR checklist：改了依赖管理方式，必须同步改 TOOLS.md。把它当 runbook 维护，而不是一次性配置文件——每次 agent 犯了环境相关的错，就补一条，成本极低，收益复利。

## 总结

TOOLS.md 的本质，是把口口相传的机器常识固化成 agent 可读的执行契约。你本来就要在对话里反复纠正它的那些环境问题，一次性写下来就好。写得短、写得准、定期验证——它比任何 prompt 技巧都更能降低 agent 的翻车率。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/220c0dd5e94483c1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/f92be707d28d52d4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/b7c510c98d650338.png)

