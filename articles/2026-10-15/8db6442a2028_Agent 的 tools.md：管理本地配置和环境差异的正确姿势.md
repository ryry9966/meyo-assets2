---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 41594
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

OpenClaw 的 agent 长期跑在固定 workspace 里，而多数人不只有一台机器：家里一台 Mac mini 常驻，一台 VPS 跑定时任务，笔记本偶尔调试。workspace 里大家熟悉的是 AGENTS.md（行为规范）和 MEMORY.md（记忆），但真正决定任务能否一次跑通的，往往是 TOOLS.md——它描述的是**这台机器**上工具的真实状态，并会随上下文注入给 agent。

## 问题

Agent 对本地环境的假设经常出错：

- 在 VPS 上写 `sed -i 's/a/b/' file`，到 macOS 报错（BSD sed 需要 `-i ''`）；
- 让它跑 `python xxx.py`，机器上只有 `python3`；
- 以为 ffmpeg 已装，实际在 `/opt/homebrew/bin` 且没进 PATH；
- 包管理器、node 版本管理器、代理端口，每台机器各不相同。

Agent 的默认策略是现场试探：先跑一次，失败，再修。单次成本不高，但任务一多，token 浪费和失败率都很可观，且不同机器上同一条指令行为不一致，排障时很难分清是环境问题还是任务问题。

## 做法

核心原则一句话：**TOOLS.md 只写这台机器的事实，不写期望。**

1. **按机器分离 workspace**。`~/.openclaw/workspace` 按主机区分（如 `workspace-macmini`、`workspace-vps`），TOOLS.md 跟随各自 workspace，天然隔离。
2. **固定小节，控制篇幅**。建议结构：系统与包管理器 / 关键工具及版本 / 路径约定 / 启停命令 / 已知坑。总量控制在 50 行内，每条给可复制的命令而非描述。
3. **写事实，附验证**。例如：`sed 是 BSD 版，-i 后必须带空串参数`；`ffmpeg: /opt/homebrew/bin/ffmpeg (7.1)`。关键条目附一条验证命令，agent 可以自查而不是盲信。
4. **沉淀排障结论**。每次 agent 排障成功，把根因压缩成一行追加进"已知坑"，这是 TOOLS.md 增值最快的方式。

## 踩坑点

- **写成文档堆砌**。把 README、安装手册全塞进去，几百行之后 agent 只会摘要扫描，关键信息被稀释，效果反而变差。
- **写"应该"而不是"是"**。"python 应该可用"等于没写，确认后应写成 `python3 (3.12)，无 python 别名`。
- **敏感信息直接入库**。token、内网 IP 写进去会随上下文被读走，用环境变量引用代替。
- **多机整份复制**。图省事把 A 机器的文件拷到 B 机器，路径互相污染，比不写还糟。
- **升级后忘了更新**。node 18 升到 22，文件里还是旧版本，agent 按旧版本选 API 直接报错。

## 可复用建议

- 写个小脚本输出机器快照（`uname -a`、`which python3`、`node -v` 等），初始化 workspace 时跑一遍作为底稿；
- 每条工具记录固定三要素：路径、版本、验证命令；
- 每周或大版本升级后花五分钟 review，删掉失效条目；
- 共用机器场景下，把"机器事实"和"个人偏好"分开，前者进版本库，后者不进。

## 总结

TOOLS.md 的价值不在文件本身，而在于把环境的隐性知识变成 agent 的显性输入。写得短、写得准、持续沉淀，agent 在每台机器上的行为就会从"大概率试探成功"变成"一次跑通"。这件事没有技术门槛，只有纪律门槛。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/d44e4e027709292a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/09904f98419f44a5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/8f6ee0c7e07509ed.png)

