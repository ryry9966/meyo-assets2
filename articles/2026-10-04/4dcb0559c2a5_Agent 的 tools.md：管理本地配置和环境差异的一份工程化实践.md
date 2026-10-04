---
title: Agent 的 tools.md：管理本地配置和环境差异的一份工程化实践
feedId: 40482
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

同一个 Agent 配置跑在多台机器上是常态：MacBook 上开发，家里 Linux 主机跑常驻任务，VPS 对外服务。系统提示一样，环境却完全不同——`ffmpeg` 一台在 `/opt/homebrew/bin`，一台在 `/usr/bin`；一台有 GPU，其余没有；部分 MCP server 只在内网机器上活着。Agent 出错最多的场景往往不是能力问题，而是**它不知道这台机器长什么样**。

## 问题

常见的错误姿势有三种：

1. **环境信息硬编码进 AGENTS.md 或系统提示**：换台机器就失真，几台机器几份漂移的副本。
2. **全靠会话里口头纠正**：“这台机器用 pnpm 不用 npm”说了一百遍，下次还错。
3. **改用 JSON/YAML 配置**：机器可读，但 Agent 需要的是上下文——这个工具在哪、什么时候用、有什么坑，纯结构化数据表达不了。

tools.md 补的就是这个缺口：一份**只描述本机事实**的 markdown，随 workspace 注入会话上下文。

## 做法

**1. 划清职责。** AGENTS.md 管行为与规则（跨机器通用），tools.md 管机器事实（本机私有）。这条边界是整套方案的地基。

**2. 模板 + 本机 overlay。** `tools.template.md` 进 git 当骨架，实际 `tools.md` 进 gitignore，各机器自行维护。新机器上复制模板填空。

**3. 固定分区，一行一个事实：**
- 环境概览：OS、包管理器、关键运行时版本
- 工具真实路径：以 `which` 的输出为准，不凭记忆写
- 常用命令的正确姿势：这台机器上验证过能跑通的那条
- 已知坑：如“node 由 nvm 管理，系统 PATH 里没有”
- 禁止操作：如“不要动 /etc/nginx，配置由面板托管”

**4. 每条事实配一条验证命令。** 写“ffmpeg 在 /opt/homebrew/bin”，就附上 `ffmpeg -version`，让 Agent 能自行复核。

**5. 定期体检。** 让 Agent 执行固定流程：读 tools.md → 逐条跑验证命令 → 报告不一致 → 更新并标日期。建议每月或大版本升级后跑一次。

**6. MCP 差异一并写入。** 哪些 server 只在哪台机器注册、哪些工具远程不可用，写清楚，省掉试错轮次。

## 踩坑点

- **写成散文**：超过一屏注意力就开始稀释，删掉所有背景介绍，只留事实和命令。
- **把密钥写进去**：tools.md 会进上下文、可能被日志记录。只写“key 存放在哪”，绝不写值。
- **过期事实比缺失更糟**：Agent 会信任文件而不是现实。条目标日期，长期未验证的强制复核。
- **改完不生效**：tools.md 在会话启动时注入，改完需新开会话或显式让它重读。

## 可复用建议

- 一条判断标准：**这条信息换台机器还成立吗？** 成立进 AGENTS.md，不成立进 tools.md。
- 用 bootstrap 脚本自动采集初版（OS、包管理器、常见工具路径），脚本生成、人工修订。
- 多机/团队之间只交换 template，不交换实体文件。
- 全文控制在 100 行内，超了说明有内容该挪去 AGENTS.md 或直接删。

## 总结

tools.md 的价值不在于写了多少，而在于**每一条都值得 Agent 读**。把它当作本机的环境契约：短、准、可验证、定期更新。环境差异永远消不掉，正确的姿势是让 Agent 在每次会话开始时，就清楚地知道自己站在哪台机器上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/eab5a456be9bb76f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/a725d8f30b0e32fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/8f70efc8bfcc88ed.png)

