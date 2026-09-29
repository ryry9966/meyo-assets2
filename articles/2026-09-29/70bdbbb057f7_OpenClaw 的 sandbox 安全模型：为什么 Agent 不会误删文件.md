---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39602
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的 Agent 默认带一组工具：读写文件、执行 shell 命令、调用浏览器和插件。这是它“能干活”的前提，也是风险的来源。模型再稳，也有概率在批量重命名、清理临时目录、拼接路径时输出一条破坏性命令——`rm -rf` 路径拼错、变量为空、把宿主机路径当成工作区路径。所以真正的问题不是“模型会不会犯错”，而是**犯错时的爆炸半径有多大**。

## 问题

很多人第一反应是靠 prompt 约束：“请不要删除重要文件”。这在工程上不可接受：提示词是概率性护栏，不是边界。正确做法是把 Agent 放进 sandbox，让“能删的东西”和“不该删的东西”在文件系统层面就分开。

## OpenClaw 的做法：四层防线

OpenClaw 的安全不是单一开关，而是叠起来的四层：

1. **进程隔离**。开启 sandbox 后，Agent 的 exec 在 Docker 容器内运行，与宿主机不共享文件系统和进程空间，容器内以普通用户身份跑，不是 root。
2. **文件系统边界**。容器只把 workspace 挂载为可写区，且挂到固定路径；宿主机其余目录要么不可见，要么只读。Agent “看得见的家”只有 workspace，`rm -rf /` 删的只是容器内的 `/`。
3. **工具与命令策略**。exec 之外的工具（browser、nodes、插件、MCP）各有独立 scope；可以配置命令黑白名单，破坏性命令走确认门——Agent 生成命令，网关拦下等人工放行。
4. **审计与兜底**。所有工具调用留日志，workspace 用 git 做版本化。就算删错，恢复成本是一次 checkout。

## 踩坑点

- **挂载过宽**。图省事把整个 home 挂进容器，sandbox 直接形同虚设。可写挂载只给 workspace。
- **容器内 root**。镜像默认 root，再配合挂载过宽就是双倍风险，务必用非 root 用户跑 agent 进程。
- **Docker socket 旁路**。为了让 Agent “能管理容器”而挂载 `/var/run/docker.sock`，等于把宿主机 root 交出去，这是最常见的越权一步。
- **MCP 开新门**。sandbox 管的是 exec 和文件工具；如果接了一个能直接读写宿主路径的 MCP 服务，等于在墙上另开一扇门。任何外部工具接入前先确认它的 scope。
- **软链接逃逸**。workspace 里若存在指向外部的 symlink，删除操作可能跟着链接走出去。构建 workspace 时不要带入外部符号链接。

## 可复用建议

- **最小可写集**：Agent 需要写什么就挂什么，其余一律只读或不挂。
- **破坏性命令单独走确认**：不要全放行，也不要全禁——全禁只会让 Agent 学会绕路。
- **定期演练**：在一个填满垃圾数据的目录里让 Agent 执行清理任务，然后核对它实际触碰过哪些路径。
- **快照是兜底不是防线**：把快照/备份当作恢复手段，而不是拦截手段。

## 总结

OpenClaw 能让 Agent 放心执行任务，靠的不是模型自觉，而是把“能力”和“边界”分开建模：进程上隔离、文件上收窄、工具上分权、行为上留痕。Sandbox 不承诺 Agent 零犯错，它承诺的是——**犯错的代价被限制在一个你能随时恢复的目录里**。这个思路对任何跑自动化 Agent 的项目都适用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/206d5ed766bcc1c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/d89b47fcd5185c35.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/7d6d676861af7678.png)

