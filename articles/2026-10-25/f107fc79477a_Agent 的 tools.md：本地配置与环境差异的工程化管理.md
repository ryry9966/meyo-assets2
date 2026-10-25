---
title: Agent 的 tools.md：本地配置与环境差异的工程化管理
feedId: 42530
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景：agent 跑在你的机器上，但它不认识你的机器

OpenClaw 的 agent 不是跑在云端沙箱里，而是跑在本地：执行 shell 命令、调你写的脚本、读写本地文件。但模型对"你这台机器"一无所知——macOS 还是 Debian、docker 要不要 sudo、node 是 nvm 管的还是系统自带、脚本放在 `~/scripts` 还是 `/opt/tools`。这些信息不在任何训练数据里，agent 只能猜，猜错就是一次失败的工具调用。

tools.md 就是为此存在的：放在 workspace 根目录、与 AGENTS.md 同级的机器级工具说明书。每次会话它会被注入 agent 上下文，本质是把"我对这台机器的了解"固化成 agent 可读的契约。

## 三种常见的坏做法

1. **塞进系统提示词或 SOUL.md**：环境说明和身份、语气偏好混在一起，越改越乱。
2. **每次对话口头解释**："注意这台机器要用 podman……"重复烧 token，还总会漏。
3. **多机共用一份配置**：笔记本上的路径抄到家用服务器上，全是坑。

## 怎么写：五步

1. **从失败倒推**。翻 agent 最近跑挂的工具调用，每条失败背后都有一条没写下来的环境事实，这是最有效的内容来源。
2. **只写命令级事实**。可复制执行的命令、确切路径、版本约束。不写"请小心处理路径"这类模型无法执行的话。
3. **按工具分节，每节三行内**：是什么、怎么调、坑在哪。例：

```markdown
## docker
- 本机 docker 需要 sudo，或把 agent 用户加入 docker 组
- 构建统一走 ~/scripts/build.sh，别直接敲 docker build
```

4. **冷启动验证**。新开会话，不加任何口头提示，直接让 agent 完成对应任务。失败即文档欠账。
5. **随环境迭代**。升级了 node、换了 shell、加了别名，顺手回填，把它当 `.zshrc` 一样对待。

## 踩坑点

- **太长**。tools.md 每次会话都占上下文，几百行会稀释关键指令。经验值：单机控制在 60 行以内，放不下的流程改写成 skill。
- **放密钥**。它会进 prompt，可能随会话日志落地。密钥走环境变量或 MCP 配置，tools.md 只写"token 从哪个环境变量读"。
- **和 SOUL/USER 混淆**。身份、说话风格、用户偏好归别的文件，tools.md 只回答"这台机器怎么干活"。
- **重复内置工具文档**。不用教 agent 怎么用内置工具；只写你自己加的脚本、别名、本机特殊行为。MCP server 的连接配置在配置文件里，tools.md 只补"装在哪、怎么调"。
- **多机同步冲突**。建议拆两份：跨机器的共享约定（命名规范、脚本入口）进版本控制；机器差异部分每台一份不同步，或按机器名分节，让 agent 只认自己那节。

## 可复用建议

```markdown
# machine profile
系统 / shell / 包管理器 / 常用路径

# tools
按工具分节：命令 + 用法 + 坑

# known pitfalls
历史失败记录，一行一条
```

- 把"冷启动测试"常态化：每周或每次大改环境后跑一次高频任务，失败即欠账。
- 划清边界：机器相关的进 tools.md，可跨机器复用的流程沉淀成 skill，别堆在一起。
- 版本管理共享部分，改环境顺手提交，回滚有据可查。

## 总结

tools.md 的价值不在"写得多"，而在"准"。短、具体、只写命令级事实，按机器隔离，随环境演进。配合定期冷启动测试，agent 就能从"猜着干活"变成"照着干活"——这是本地自动化稳定性的最低成本投入。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/1922dd8b458169bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/1ff0e25ab8f7d660.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/d6c718baaf6efb64.png)

