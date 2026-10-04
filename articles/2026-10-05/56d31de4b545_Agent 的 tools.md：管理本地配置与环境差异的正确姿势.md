---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 40486
source: 综合讨论
publishedAt: 2026-10-05
---

# Agent 的 tools.md：管理本地配置与环境差异的正确姿势

## 背景

跑 Agent 久了，多数人会经历同一个阶段：一台 MacBook、一台 homelab、一台 VPS，各挂一套 OpenClaw。Skills 和 MCP 工具是共享的，但每台机器的真实环境并不一样——装没装 ffmpeg、Python 是谁管的、docker daemon 在不在本机、路径长什么样。这些信息 Agent 默认一无所知，只能猜。

## 问题

猜的代价很具体：

- 在没有 ffmpeg 的机器上编排了一条依赖 ffmpeg 的流水线，跑到一半才报错；
- 把 `/Users/xxx/...` 的绝对路径带去了 Linux 机器；
- 同一个工具不同版本，参数行为不一致；
- 环境信息散落在记忆文件、skill 描述和聊天记录里，三处互相漂移。

根因只有一句话：环境事实缺少一份 Agent 可读的、每台机器唯一的 source of truth。

## 做法

原则先立住：指令放 AGENTS.md 和 skill 文件，环境事实放 tools.md。一份 tools.md 只属于一台机器，不做多机同步。

步骤：

1. 每台机器建 `~/.config/agent/tools.md`（或放工作区根目录），固定六个小节：主机标识、OS 与包管理器、工具清单、路径约定、服务与端口、禁区。
2. 半自动生成工具清单：一个十几行的脚本遍历常用命令，输出 `command -v` 和 `--version` 结果粘进文件，再手写补上注意事项。
3. 在 AGENTS.md 启动区加一行：「执行环境相关操作前先读 tools.md；内容与实际不符时，以实际命令输出为准，并提醒我更新文件。」
4. 建立更新闭环：Agent 每次发现环境不符（工具缺失、版本变化），追加到「变更记录」小节并标注日期。

文件大致长这样：

```markdown
# tools.md — macbook-m3（最后核对 2025-06-20）
## 工具
- ffmpeg 7.1 @ /opt/homebrew/bin/ffmpeg
- node 22.x（nvm 管理），默认 22
- python 3.12（uv 管理），无系统 pip
## 路径
- 工作区：~/works/agent
## 服务
- docker context 指向 homelab，本机无 daemon
## 禁区
- 禁止全局 pip install，一律走 uv
```

控制在几十行以内。标准是你愿意持续维护的程度，不是完备性。

## 踩坑点

1. **写进密钥**。tools.md 会被加载进上下文，只写指针（环境变量名、「密钥在密码管理器某条目」），绝不写值。
2. **长成第二份 AGENTS.md**。一旦混入工作流指令，两个文件必然漂移。事实归 tools.md，怎么做事归别处。
3. **陈旧信息比没有更糟**。必须配两条规则：文件顶部标注最后核对日期；关键操作前 Agent 用 `command -v` 实测，不盲信文件。
4. **用 git 把一份 tools.md 同步到多台机器**。这直接违背设计初衷。放 dotfiles 做 per-host include，或者干脆 gitignore。
5. **写了但 Agent 不读**。检查那行引导指令是否真的在启动路径上，而不是躺在某个没人引用的文档里。

## 可复用建议

- 六节模板（主机标识 / 工具 / 路径 / 服务 / 禁区 / 变更记录）直接抄。
- 「只写事实，不写指令」是唯一硬约束。
- 脚本生成的段落用围栏代码块包住，下次重跑 diff 一下就知道环境哪里变了。
- 超过 60 行就按领域拆成子文件，主文件只留索引。

## 总结

tools.md 解决的不是高深问题，而是一类高频事故：Agent 对环境的错误假设。一台机器一份、只写事实、标注日期、运行时验证——四条做完，「环境靠猜」就变成了「环境有据可查」。成本半小时，回报是少修很多次跑到一半才挂掉的流水线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/dce808c4712cb13f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/1cdaa8cc502258e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/ffec5efc2c1bffe3.png)

