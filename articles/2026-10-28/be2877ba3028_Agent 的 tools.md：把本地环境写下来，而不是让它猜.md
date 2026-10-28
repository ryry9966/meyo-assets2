---
title: Agent 的 tools.md：把本地环境写下来，而不是让它猜
feedId: 42851
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

跑 Agent 做自动化的人大多遇到过这类场景：同一个任务，在自己机器上一次跑通，换到服务器上 Agent 先花三轮试探 `grep` 还是 `rg`、Python 在不在 venv 里、包管理器是哪个。Agent 对本地环境没有记忆，它只会猜——猜错就重试，重试就烧 token 和时间。

tools.md 的定位很简单：一份给 Agent 读的本地环境清单。它不是项目文档，而是把"这台机器上什么可用、怎么用、哪些别碰"写成事实声明，让 Agent 少做环境考古。

## 问题

实际使用中，环境信息散落在四处：shell rc 文件、`.env`、项目 README，以及每个人脑子里那句"我这台机器有点特殊"。Agent 每次会话都要重新发现一遍。更麻烦的是系统性差异：

- macOS 和 Linux 的 `sed`、`date` 行为不同；
- 交互式 shell 里的 alias，在 Agent 实际执行的非交互命令里根本不存在；
- MCP 服务的启用状态每台机器不一样，但团队共享的 AGENTS.md 没法覆盖个人差异。

## 做法

按四步搭起来：

1. **盘点**。先让 Agent 跑一次环境探测（或自己写个小脚本），列出实际可用的二进制、版本、关键路径。以探测结果为准，不凭记忆写。

2. **分层放置**。建议两层：用户级清单（如 `~/.openclaw/` 下的个人配置目录）放本机事实，不进仓库；项目级 `tools.md` 放团队共识，进仓库。项目级覆盖用户级，后加载优先。

3. **写事实，不写散文**。每条尽量一行、可验证：

   ```text
   - rg 14.x：优先于 grep
   - python：/opt/venv/bin/python，别用系统 python
   - macOS：sed -i 需要 '' 占位参数
   - MCP：本机启用 filesystem 和 git server，启动命令见启动脚本
   ```

   再加一节"禁令"：不许动的目录、不许执行的命令。

4. **带校验**。清单末尾放一条自检命令（版本断言脚本即可），标注"最后验证于某版本/日期"。环境变了就重跑。

## 踩坑点

- **写成文档**。tools.md 每个字都会进上下文，写成三页散文等于付费让模型读小说，控制在几十行。
- **写了 alias**。Agent 执行命令通常不走交互式 shell，alias 大概率不生效，写二进制全名或绝对路径。
- **版本漂移**。清单写着 Python 3.11，系统已升到 3.12。没有自检的 tools.md 三个月后就是错误信息源。
- **和项目文档重复**。环境事实放 tools.md，业务知识放 README/AGENTS.md，重复内容迟早不同步。
- **个人路径进了共享仓库**。绝对路径、本机 venv 位置只放用户级那份。
- **写入密钥**。tools.md 会被完整注入上下文，任何 secret 都不该出现。

## 可复用建议

- 把探测脚本固化，一条命令重新生成清单，tools.md 改为半自动维护。
- 加一节"已知差异"，按系统分列（macOS / Linux / WSL），新机器接入时对照补齐。
- 团队新增开发依赖时，把更新 tools.md 列为 PR 检查项，和代码一起评审。
- 条目按"高频使用优先"排序，注入和检索都更省。

## 总结

tools.md 解决的不是功能问题，而是确定性问题：把"Agent 每次猜环境"变成"每次读清单"。原则就三条——写事实不写散文、个人与团队分层、带自检防漂移。把它当代码维护，Agent 的重试次数和 token 账单会诚实地反映出来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/f20f303c3471d42d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/b265bb45b2f3df1c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/9c70913d4e7b1711.png)

