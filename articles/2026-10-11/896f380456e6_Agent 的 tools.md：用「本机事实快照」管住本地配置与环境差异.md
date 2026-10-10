---
title: Agent 的 tools.md：用「本机事实快照」管住本地配置与环境差异
feedId: 41158
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

OpenClaw 的工作区里，`AGENTS.md` 管行为规范，`SOUL.md` 管人格设定，而 `TOOLS.md` 是最容易被写歪的一份——它的定位是给 Agent 看的**本机事实快照**：这台机器上有什么工具、怎么调用、有什么坑。

典型场景：我在笔记本（macOS）和一台小主机（Ubuntu）上各跑一个 OpenClaw，工作区模板一致，但环境完全不同。Agent 落在哪台机器上干活，就得靠哪台的 tools.md 校准行为。这份文件写得好坏，直接决定 Agent 是"一次命中"还是"反复试错"。

## 问题

实践中翻车最多的三类：

1. **路径和事实写死**。`/Users/xxx/venv/bin/python` 这种绝对路径，换机器即失效；Agent 基于过时事实给出的命令必然出错。
2. **敏感信息入库**。API key 直接写进 tools.md，随后被备份、同步、截屏分享，一路泄漏。
3. **文档与现实漂移**。写着"ffmpeg 可用"，机器上根本没装；Agent 信以为真，连错三轮才回退到别的方案。

还有个隐性成本：有人把它写成人类向的运维 wiki，几千字。Agent 每次会话都要读，挤占上下文，反而抓不住重点。

## 做法

我的模板只保留三节，控制在一屏以内：

```markdown
# TOOLS.md —— 本机：小主机，Ubuntu 22.04

## 环境事实
- shell: bash，包管理: apt
- 无 GPU；转码类任务走 CPU，不要建议 CUDA 方案

## 工具清单
### ffmpeg
- 用途：音视频转码、抽帧
- 校验：command -v ffmpeg
- 备注：4.4 版本，部分新参数不支持

## 禁区
- 不要直接改 /etc/nginx；改 ~/repos/nginx-conf 后跑 reload.sh
```

配套几条原则：

1. **分层**。机器事实进 tools.md，项目依赖交给各仓库自己的 AGENTS.md，跨机器的通用习惯写进工作区规范。三者不要互相渗透。
2. **只写名，不写值**。环境变量写名字和用途（如 `TRANSCRIBE_API_KEY`，从 shell profile 读取），值永远不落在工作区文件里。
3. **校验命令必须只读**。用 `command -v`、`--version` 这类幂等谓词，别写 `apt install`——否则 Agent 会自作主张去装东西。
4. **路径锚定工作区**。写 `./scripts/backup.sh`，不写绝对路径。
5. **半持久能力也入清单**。MCP server、浏览器插件这类不常驻的能力，写清启动方式和端口，省得 Agent 猜。
6. **让 Agent 自己对账**。在 HEARTBEAT 指令里加一条：周期性执行 tools.md 里的校验命令，失败项标记，并在下一次会话提醒更新。

## 踩坑点

- **校验写成安装**：如上，只读谓词是底线，血的教训。
- **贪多**：工具清单超过十项后，Agent 选工具的准确率肉眼可见地下降。只放高频项和易错项。
- **多机共用一份**：清单一旦共享，就没人敢删自己机器上没有的条目，文件只会膨胀。每台机器独立一份。
- **忘了更新**：装新工具、卸载旧工具都不记。heartbeat 对账能兜底，但最好养成动手后顺手 diff 的习惯。

## 可复用建议

- 把 tools.md 模板放进自己的 dotfiles，新机器初始化工作区时三分钟填完。
- 「环境事实」固定四行：OS、shell、包管理、GPU 有无，够用。
- 改动机器环境后 `git diff` 工作区，把 tools.md 当代码对待，进 PR 评审。
- 团队内统一「禁区」的写法——这台机器上 Agent **不能碰什么**，往往比它能做什么更重要。

## 总结

tools.md 不是配置中心，也不是文档，而是**给 Agent 的本机事实快照**。三条纪律贯穿始终：机器事实与项目事实分层、只写名不写值、校验只读且定期对账。做到之后，同一套 OpenClaw 配置在笔记本、小主机、VPS 之间迁移，基本不会再出现"这台机器上不行"的对话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/edcef7c43d61a13a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/189a0305343cdce0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/87cca0554197792e.png)

