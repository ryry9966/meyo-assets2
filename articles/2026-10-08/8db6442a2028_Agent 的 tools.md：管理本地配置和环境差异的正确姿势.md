---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 40896
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

Agent 跑起来之后，工具和技能通常是可移植的：MCP server 换台机器还能连，插件清单跟着仓库走。但真正不可移植的是那台机器本身——shell、包管理器、Python 路径、代理、端口，每台都不一样。OpenClaw 的 tools.md 就是干这个的：给当前 workspace 挂一份“环境事实清单”，Agent 每次会话都会读。它不是文档，是上下文的一部分。

## 问题

没有这份清单时，Agent 靠两件事补齐环境信息：探测和猜测。探测就是跑一堆 `which`、`uname`，浪费 token 和时间；猜测更糟——在 dnf 机器上敲 `apt`，把依赖装进系统 Python，忘了代理导致下载超时，都是真实发生的失败。更麻烦的是，系统提示词和工作区说明是跨机器共享的，把“我家里这台用 zsh”写进去，到别的机器就成了噪音，甚至误导 Agent。

## 做法

按“共享层 + 本地层”来组织：

1. **共享层**：workspace 里建 tools.md，只写对所有机器成立的约定——测试怎么跑、哪些目录是禁区、服务默认端口。进版本控制，团队共用。
2. **本地层**：每台机器写一份 tools.local.md（或明确标记的独立段落），只写这台机器的事实，加进 .gitignore。
3. **只写事实，每条一行，标明适用范围**。参考骨架：

```markdown
# tools.local.md
- shell: zsh, macOS arm64
- python: /opt/homebrew/bin/python3.12（不要用系统 python）
- proxy: 终端走 127.0.0.1:7890，git 单独配置
- 禁区: ~/data 是原始数据，只读
```

4. **校验**：写完让 Agent 先跑一遍探测，比对 tools.md 与实际情况，不一致的当场改掉。
5. **反哺**：Agent 探测失败后自行纠正的结论，追加回 tools.md。失败一次，沉淀一条，这是它最容易被忽略的价值。

## 踩坑点

- **写成散文**。tools.md 每次会话都进上下文，两百字的事实比两千字的教程有用，Agent 也会对长文件跳读。
- **把密钥写进去**。哪怕只在本机，也很容易顺手提交。密钥走环境变量，tools.md 里只写机制不写值。
- **层级混乱**。机器特有的事实进了共享文件，别的机器上的 Agent 会信以为真，排查起来非常绕。
- **信息过期**。写的是 Python 3.10，机器已升到 3.12，Agent 会信文档不信探测——比不写还糟。条目越少，过期面越小，每条尽量写成“可被一句话证伪”的形式。
- **职责重叠**。环境事实归 tools.md，行为规范归 AGENTS.md，别两边都写。

## 可复用建议

- 把 tools.md 当“环境快照”维护：升级系统、换包管理器之后主动 diff 一次，而不是等 Agent 报错再想起来。
- 多人协作时，共享层只放团队共识，个人差异全部下沉到本地层，避免代码评审里出现“你机器的事”。
- 新机器接入时，先让 Agent 读 tools.md 并报告可疑项，作为初始化检查步骤，能提前暴露大半配置问题。

## 总结

tools.md 的价值不在于写得多全，而在于把 Agent 最容易猜错的那部分环境事实，用最少的 token 固定下来。共享层管约定，本地层管差异，探测做校验，失败做反哺——四个动作，够用了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/407e0fa9bceaa8ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/ecaf44c5d8a74dd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/19c5fb39ef41af40.png)

