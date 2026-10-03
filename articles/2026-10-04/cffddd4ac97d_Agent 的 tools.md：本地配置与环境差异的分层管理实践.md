---
title: Agent 的 tools.md：本地配置与环境差异的分层管理实践
feedId: 40304
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

OpenClaw 的 Agent 每次会话开始都会读取一份环境说明，tools.md。它的作用类似 manifest：告诉 Agent 这台机器上有什么工具、装在哪、版本是什么、哪些事做不了。听起来简单，但实践中多数团队要么不写，要么写成一坨没人维护的流水账，Agent 只能靠猜。

## 问题

靠猜的代价在多环境场景下集中爆发：

1. **包管理器误判**。在 macOS 上执行 `apt-get`，在 Ubuntu 上执行 `brew install`，装一半失败后 Agent 开始"自救"，往系统里塞一堆来源不明的包。
2. **路径漂移**。同一个项目在开发机是 `/Users/me/work/proj`，在容器里是 `/workspace/proj`。文档里写死一个，另一台必挂。
3. **文档腐化**。tools.md 写完当天是准的，两周后依赖升级、目录重构，没人回头改。Agent 信任过期文档，报错，然后自作主张地"修复"环境——越修越乱。

根因只有一个：把"所有机器共享的事实"和"这台机器独有的差异"混在一个文件里，且没有验证机制。

## 做法

分两层，各管各的：

**tools.md（进 Git，团队共享）**：只写所有环境都成立的事实。
- 工具链与版本下限（如 `node >= 20`）
- 项目通用的构建/测试命令
- 明确禁止事项（如"不要全局 pip install"）

**tools.local.md（gitignore，每台机器一份）**：只写差异。
- 包管理器、二进制路径
- 本机不可用的能力（如"无外网""无 sudo"）
- 硬件相关备注（GPU、内存限制）

每条记录遵循固定格式：**事实 + 验证命令**。例如：

```markdown
## runtime
- node >= 20；验证：`node -v`
- 包管理器：pnpm（不要用 npm/yarn）；验证：`pnpm -v`
```

再配一个 20 行的 bootstrap 脚本，首次部署时嗅探环境（shell、包管理器、关键路径），生成 tools.local.md 初稿，人工过一遍确认。写文档的成本从半小时降到两分钟。

步骤小结：

1. 建立基线 tools.md，进版本库
2. 写 tools.local.md 模板 + 嗅探脚本
3. `.gitignore` 加上 `tools.local.md`
4. 在 Agent 系统提示里声明读取顺序：local 覆盖 baseline

## 踩坑点

- **别把秘密写进去**。API key、内网地址一律进 `.env` 或凭据管理。tools.md 会被 Agent 全文读进上下文，也会被截图、被同步。
- **别写成 runbook**。tools.md 回答"这台机器有什么"，不回答"怎么部署"。操作手册放别处。
- **控制在 60 行以内**。这个文件每次会话都进上下文，写得越长，留给干活的空间越少。Agent 自己能探测到的（当前目录、系统版本）不用写。
- **明确覆盖规则**。local 和 baseline 冲突时听谁的？必须写死：local 优先。否则 Agent 遇到矛盾会随机站队。
- **限制 Agent 的写权限**。允许它往 tools.local.md 记录自己装了什么，但禁止改 baseline；baseline 变更必须走人审。

## 可复用建议

- 把 tools.md 当代码：改动走 PR，diff 能直接看出环境策略的漂移。
- 每周让 Agent 跑一次环境审计：逐条执行验证命令，输出"文档 vs 现实"的差异报告，作为 drift 告警。
- 新成员或新容器入场的验收标准就一条：Agent 读完后能正确回答"用哪个包管理器、项目在哪、什么不能做"。

## 总结

tools.md 本质是 Agent 和机器之间的契约。契约要小、要分共享与本地两层、要可验证。做到这三点，Agent 在开发机、服务器和容器里的表现才会一致，不是靠它聪明，而是靠你把差异提前说清楚了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/2a43b7316e0ba4b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/f6b4086d76a774f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/d92405c7d20f3f8e.png)

