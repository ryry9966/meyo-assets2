---
title: tools.md 分层实践：让 Agent 不再猜你的本地环境
feedId: 41319
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景

Agent、MCP server 和各类插件真正跑起来时，面对的执行环境千差万别：home 目录不同、Python/Node 版本不同、内网网关不同、甚至同名命令指向不同二进制。模型对环境的"想象"经常是错的，而它唯一的纠错来源是我们写给它的上下文。tools.md 就是这个上下文的落点：一份告诉 Agent "这台机器、这个项目里工具到底长什么样"的约定文件。

## 问题

常见的三种崩法：

1. Agent 按默认路径找配置（`~/Library` vs `~/.config`），找不到就开始瞎猜；
2. 同一份 tools.md 提交进仓库后，在同事机器上全是错的——路径、端口、虚拟环境名都不一样；
3. 敏感信息（token、内网地址）要么没写导致 Agent 连不上服务，要么写了直接进 git 历史。

根因是同一个：把"团队共识"和"本机事实"混进了同一个文件。

## 做法

按职责拆成三层，每层只干一件事：

1. **全局层** `~/.openclaw/tools.md`：跨项目通用的本机事实——默认 Python、包管理器、代理、shell 类型。跟随用户，不入库。
2. **项目层** `tools.md.example`（提交）：只写团队共识——用到哪些 MCP server、启动入口、目录约定、版本约束。不含任何机器相关值。
3. **本地层** `tools.md`（gitignore，从 example 复制）：机器相关值。新环境 setup 就是 `cp tools.md.example tools.md` 再改几行。

```
# tools.md（本地层示例）
python: /opt/homebrew/bin/python3.12
venv: .venv-gpu
api_base: http://127.0.0.1:8787
secrets: 见 .env，不入库
```

Agent 注入上下文时按 全局 → 本地层 → example 兜底 的顺序合并，冲突时后者覆盖前者。文件头加一行注释标注生成时间和 hostname，方便排查过期。

## 踩坑点

- **把 secrets 写进 tools.md**。它不是 .env 的替代品。只写"去哪取"（如 "token 见 .env 的 XXX_KEY"），不写值本身。
- **写太多**。tools.md 会被注入上下文，300 行流水账等于烧 token。只写 Agent 决策真正依赖的差异项，其余让它现场探测（`which`、`ls`）。
- **文档腐烂**。tools.md 描述的是事实，事实会变。约定：改环境的 PR 必须同步改 example；CI 里加个 lint 脚本校验关键字段存在。
- **当指令文件用**。"执行前先问我"这类行为规则放 AGENTS.md 或系统提示，tools.md 只承载环境事实。混写会让两层职责漂移，最后谁都不敢删。

## 可复用建议

- 配一个 `doctor` 脚本：读取 tools.md，逐项 `test -x` / 探活 URL，Agent 启动前先跑一遍，把"文档说的"和"机器现实"对齐。
- 条目尽量写成可验证格式（路径、版本号、URL），少写形容词。
- 检验 example 设计得好不好，看新人 onboarding 成本：理想情况 ≈ cp + 改 3 行。

## 总结

tools.md 的本质是给 Agent 一份诚实的环境快照：**共识入库、事实留本地、密钥进 .env，分层合并、定期对账**。做到这几点，Agent 在不同机器上的行为差异就从"玄学"变成了一个可以 diff、可以 lint 的配置问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/7cc932ace31f5697.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/431f61e0b71e82c9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/58ae94d9d4297c2e.png)

