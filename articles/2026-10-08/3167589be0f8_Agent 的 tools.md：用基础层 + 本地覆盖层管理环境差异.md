---
title: Agent 的 tools.md：用基础层 + 本地覆盖层管理环境差异
feedId: 40866
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

Agent 跑得久了，每个环境都会积累出一堆"只有这台机器知道"的事实：MCP server 的端口、Python 解释器路径、代理地址、有没有 GPU、某个脚本放在哪里。这些东西散落在 prompt、启动脚本和会话记忆里，换台机器或换个目录就全对不上。

tools.md 的初衷很简单：给 Agent 一份它能在会话开始时读到的工具说明。但如果把它当成"什么都往里写"的文件，很快就会变成跨机器不可复制的负担。

## 问题

几个典型症状：

- 在公司电脑调好的 agent，回家跑第一件事就是找错解释器路径；
- 同事拉了仓库，tools.md 里的绝对路径、内网地址全部失效，只能手工改，改完还容易误提交；
- tools.md 三个月没更新，Agent 按它调用工具，报错是 `file not found`。

根因只有一个：把**"工具是什么"（稳定）**和**"这台机器上工具在哪、怎么连"（易变）**写在了同一个文件里。

## 做法：base 层 + local 覆盖层

把 tools.md 拆成两层：

1. **base（入库）**：只写环境无关的事实——有哪些工具、干什么用、怎么调用、依赖什么前置条件、引用哪些环境变量**名**。
2. **local（不入库）**：只写机器相关的值——路径、端口、代理、是否启用某工具。命名为 `tools.local.md`，加进 `.gitignore`，同时提交一份 `tools.local.example.md` 当模板。

加载顺序约定死：**base 先读，local 覆盖 base，都没有就回落默认值**。把这条优先级写进 agent 的启动指令，别让它自己猜。

每个工具一节，可抄的结构：

```markdown
## tool: shell-exec
- 用途：执行本地命令
- 调用：mcp shell-exec
- 前置：PATH 中有 bash
- 校验：`bash -c 'echo ok'`
- 本地差异：见 tools.local.md 的 shell 段
```

关键是"校验"这一行：给每个工具一条**会话开始就能跑的自检命令**。这样 tools.md 从"可能过期的散文"变成"可执行的文档"——Agent 开场先跑一遍自检，环境不对当场暴露，而不是调用到一半才炸。

## 踩坑点

- **base 里混进绝对路径**：最常见的污染源。规则定死：base 出现 `/Users/` 或 `C:\` 就过不了 review。
- **两层写重复信息**：同一个端口 base 和 local 各一份，改了 local 忘了 base。原则：一个事实只在一层出现。
- **把密钥写进 tools.md**：只写变量名和来源（"读 env 里的 XX_TOKEN"），永远不写值。tools.md 会被读进上下文，等于进了日志。
- **文件越写越长**：tools.md 每个字都占上下文预算。超过一屏就该合并工具、砍掉 Agent 用不上的背景说明。
- **跨平台细节**：macOS/Linux 的路径分隔、shell 差异，在 local 里顺手注明平台，别让 Agent 猜。

## 可复用建议

- 把 tools.md **当代码对待**：进 PR、被 review、有变更记录，而不是随手维护的备忘录；
- 每个 repo 一份 base；个人偏好（编辑器、通知方式之类）放机器级全局文件，别混进项目；
- 自检命令保持廉价：亚秒级、无副作用，跑十次也不心疼；
- 给 local 文件丢失留兜底：example 模板 + 报错信息里明确提示"缺哪一项、去哪补"。

## 总结

tools.md 的正确姿势不是"写得更全"，而是"分层更对"：稳定事实入库，环境差异进本地覆盖层，再用自检命令让文档保持诚实。做完这三件事，Agent 换机器、跨成员协作时那些"环境玄学"问题，基本就消失了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/042b298a35d8170b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/fc79a8a4d54d4775.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/bdbf7e3a064f899b.png)

