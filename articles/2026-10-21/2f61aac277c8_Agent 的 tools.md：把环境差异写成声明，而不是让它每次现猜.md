---
title: Agent 的 tools.md：把环境差异写成声明，而不是让它每次现猜
feedId: 42223
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

Agent 跑在本地的这段时间，最典型的故障已经不是"模型不会写命令"，而是"命令在你这台机器上不对"。macOS 的 `sed` 和 Linux 行为不一致，Python 可能是 pyenv 管的 3.12，公司网络要过代理，某个 MCP server 只在固定一台机器上装了。模型的世界知识是通用的，你的机器是具体的——这中间的落差，每次会话都在靠试错填补。

## 问题

常见症状：

- Agent 写 `python3 -m venv`，实际机器上只有 pyenv 的 `python3.12`；
- 在 macOS 上生成 GNU 风格的 `sed -i`，直接报错；
- 团队里每个人在自己的 prompt 里口头描述一遍环境，说法还不统一。

根因只有一个：**环境事实没有被声明，而是被模型根据先验猜测。**

## 做法

tools.md 就是把这台机器"能用什么、怎么用、不许碰什么"写成一份短文档，Agent 启动时读取。

**1. 先盘点，再动笔。** 花五分钟跑一遍 `which python3 node docker`、`python3 --version`，确认代理端口和镜像源，把真实结果记下来，别凭记忆写。

**2. 分层存放。**

- 全局层：`~/.openclaw/tools.md`，写机器级事实：OS、shell、代理、全局 MCP server；
- 项目层：项目根目录 `tools.md`，写项目级事实：启动命令、端口、目录约定、禁忌操作。项目层覆盖全局层。

**3. 只写声明式事实。** 每条一行，例如：

```markdown
- python: pyenv 管理，用 `python3.12`，不要直接 `pip install`
- 网络: 走 http://127.0.0.1:7890，npm/pip 已配内部源
- MCP: filesystem 常驻；playwright 仅本机可用
- 禁止: 不改 ~/.ssh 下任何文件，不对绝对路径执行 rm -rf
```

**4. 治理版本。** 项目层进 git，个人机器差异放 `tools.local.md` 并 gitignore；全局层不进库。

**5. 验证加载。** 写完后让 Agent 复述一遍它理解的环境，或跑一个只读任务，确认它真的读到了——而不是假设它读到了。

## 踩坑点

- **写成小作文。** tools.md 吃的是上下文预算，散文式描述信息密度低，坚持条目化。
- **和 MCP 工具描述重复。** 工具自身能力让工具描述去讲，tools.md 只写"这台机器上的差异"，否则工具一升级两边都过期。
- **塞密钥。** key 一律走环境变量，tools.md 里只写"TOKEN 从 env 读取"。
- **只增不删。** 过期条目比没有更糟，Agent 会信以为真。换机器、升版本时顺手清理。
- **硬编码绝对路径。** 写相对项目根的路径，或用 `~` 展开。

## 可复用建议

- 维护一个五段模板：运行时 / 网络 / MCP / 目录约定 / 禁忌，新项目复制了改；
- 写一个十行的 bootstrap 脚本，自动探测版本和路径生成草稿，人工只补差异部分；
- 把 tools.md 当代码对待：进 PR、写 changelog，同事换机器时它就是现成的交接文档。

## 总结

tools.md 的价值不在文件本身，而在于把"环境差异"从运行时的试错，前移成启动时的声明。成本十分钟，收益是每次会话少烧的 token 和少走的弯路。先从全局层写下三条最痛的事实开始，比追求一次写完美更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/f9c78c5aa5642f91.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/d1a6855461f948da.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/d6e1d6a0564e69b6.png)

