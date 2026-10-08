---
title: Agent 的 tools.md：分层与可验证的本地配置管理实践
feedId: 40951
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

跑 OpenClaw / MCP 这类 Agent 时，tools.md 是 Agent 认识本机环境的“地图”：有哪些工具、路径在哪、版本要求、项目约定。单机自用，随手写一份就能跑。但只要出现第二种环境——公司电脑和家里服务器、macOS 和 Linux VPS、你和同事的机器——这份地图就开始说谎。

## 问题

实际踩到的三类故障：

1. **路径写死**。`/Users/xxx/dev/venv` 换到 Linux 直接失效，Agent 反复重试直到放弃。
2. **工具漂移**。tools.md 写着用 `docker compose`，但某台机器装的是 podman。Agent 按文档执行报错后，开始“自由发挥”。
3. **文档陈旧**。升了 Node 版本、换了 shell、挪了目录，tools.md 没跟着改。Agent 拿着过期地图导航，比没有地图更糟。

根因只有一个：把 tools.md 当静态文档，而环境是动态的。

## 做法：分层 + 可验证

核心思路两条：**共享信息与本机信息分层**，**每条描述都能被脚本验证**。

**1. 拆成两个文件**

- `tools.md`：进仓库，只写跨机器为真的内容——项目约定、通用流程、构建入口。
- `tools.local.md`：gitignore 掉，只写本机事实——路径、版本、怪癖。
- 优先级约定：local 覆盖 shared。

**2. 每个工具条目固定四行**

```markdown
## node
- 用途：脚本执行与构建
- 验证：node --version，需 >= 20
- 约定：只用 npm，不引入 pnpm
- 兜底：缺失时改用 deno task
```

“验证”和“兜底”两行最关键：Agent 失败时知道往哪退，而不是瞎猜。

**3. 写一个 doctor.sh**

遍历 tools.md 里的“验证”命令逐条执行，输出与文档不符的 diff，几十行 shell 就够。两种跑法：环境变更后手动跑；或在 Agent 启动流程里先跑一遍，把 diff 喂给 Agent，让它顺手更新 tools.local.md——生成优先于手写。

**4. 启动顺序写进引导提示**

明确告知 Agent：先读 tools.md，再读 tools.local.md，冲突以 local 为准。不写这句，它大概率只读第一个文件。

## 踩坑点

- **写成散文**。大段描述 Agent 会漏读。短条目、命令优先，每条不超过 5 行。
- **和 MCP 配置重复**。MCP server 列表在配置文件里已经有一份，tools.md 再抄一遍必然漂移。tools.md 只写“怎么用、什么时候用”，不写“有哪些”。
- **绝对路径**。统一用 `~` 或项目根相对路径；实在避不开，集中放进 local 文件。
- **忘了 gitignore**。local 文件里的密钥路径被推上仓库。模板预置 .gitignore 条目，CI 加一道检查。
- **太长**。整文件控制在 150 行内。超了说明有内容 Agent 根本用不到，砍掉。

## 可复用建议

- doctor 脚本加 `--init` 模式，直接从机器现状生成 tools.local.md 初稿。
- 条目加一个“校验时间”字段，一眼看出哪份文档已经陈旧。
- 团队场景把 tools.md 纳入 review，规则绑定：改 Dockerfile / CI 的 PR 必须同步检查 tools.md，否则环境和文档立刻分叉。

## 总结

tools.md 的本质是给 Agent 的环境接口文档。管好它靠三件事：**分层**（共享与本机分离）、**可验证**（每条都有对应检查命令）、**够小**（只写影响 Agent 决策的内容）。做到这三点，换机器、跨团队时，Agent 才不会拿着过期地图走错路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/97f495daa5e11abe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/dae45fd63c2fa36d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/9607bb1d3ff1fab7.png)

