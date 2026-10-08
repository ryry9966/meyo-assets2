---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40952
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

Agent 好不好用，一半取决于它对“这台机器”的了解程度。同一套 MCP server、CLI 插件和自动化脚本，在我的机器和同事的机器上路径不同、包管理器不同、端口占用不同，甚至一个要走代理一个不用。我过去的做法是把这一切塞进系统提示词，或者每次手动解释——结果是换台机器、来个新人、跑一次 CI，agent 就开始瞎猜路径、用错命令。

tools.md 的思路很简单：把“环境事实”写成一份 agent 会读的 Markdown，分层维护，可验证、可版本化。

## 常见问题

- agent 猜环境：用 `python` 还是 `python3`，brew 还是 apt，项目在第几块盘；
- 环境知识只存在于某个人的脑子里，不可迁移；
- 所有说明堆进 system prompt，token 贵，还经常过期；
- 图省事把 API key 明文写进提示词，日志一拉全泄露。

## 做法：三层结构

1. **全局层** `~/.config/agent/tools.md`：机器级事实。OS、shell、版本管理器（fnm/pyenv）、代理端口、Docker 是否常驻。
2. **项目层** `<repo>/tools.md`：随仓库提交。启动/测试命令、默认端口、依赖的服务、MCP server 的启动方式（本机 npx 还是容器）。
3. **私有层** `tools.local.md`：进 .gitignore。覆盖本机差异：改过的端口、内网地址、密钥位置。

优先级写死在文件第一行：私有层 > 项目层 > 全局层。

写作四原则：

- **只写事实，不写愿望**。“包管理器是 pnpm”可以，“请写优雅的代码”请放到别处。
- **每条配置附验证命令**。例如“node 由 fnm 管理（验证：`fnm current`）”，agent 先验证再动手。
- **秘密外置**。只写“密钥在 .env，用环境变量引用”，绝不写明文。
- **控制篇幅**。单文件 60 行以内，它是每次会话都要进上下文的东西。

## 踩坑点

- 私有层忘了 gitignore，token 跟着提交进仓库。加一道 pre-commit 扫描兜底。
- 全局层和项目层冲突，agent 行为飘忽。定死优先级，冲突时项目层说了算。
- 改了文件名，agent 的加载配置没同步，等于白写。改完跑一次，确认它真的读到了。
- 内容过期比没有更糟。把“验证命令失败 → 提示更新 tools.md”写进团队 review 清单，或挂个十行的 CI lint 逐条跑验证命令。
- Windows 和 Unix 路径混写，跨平台同事直接踩坑。单独开一节写平台差异。
- 把它当 README 用，塞满业务说明——它只回答“这台机器/这个项目怎么跑起来”。

## 可复用建议

- 固定小节模板：Environment / Toolchain / Run & Test / Ports / Secrets / Known Quirks，团队写起来不费脑子。
- 验证脚本 + agent 启动钩子：启动前跑一遍校验，失败项直接回传给 agent，让它自己决定降级方案还是提醒你修。
- 换新机器时先复制旧 tools.md 再改，比从零回忆快得多。
- 团队铁律一条：动工具链必须同步动 tools.md，与改代码同优先级。

## 总结

tools.md 的本质，是把环境知识从“口头 + 上下文”变成版本化、分层、可验证的事实清单。少写指令、多写事实、附验证命令、秘密外置、保持短小——这五条做到，agent 在你的机器、同事的机器和 CI 之间的行为差异会明显收敛，排障时间也会跟着降下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/5c5c5b2ea656fe88.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/13a68a72b22c4bfb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/a488c3f958051d2b.png)

