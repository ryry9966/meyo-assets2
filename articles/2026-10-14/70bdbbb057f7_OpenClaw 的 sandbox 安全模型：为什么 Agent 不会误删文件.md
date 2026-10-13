---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41467
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

OpenClaw 的 Agent 拿到的是一套真实工具：`exec` 可以跑 shell，文件工具可以直接读写工作区。在自动化场景下（cron 任务、群消息触发、无人值守的长任务），模型一旦在路径、通配符或上下文上产生幻觉，一条 `rm -rf` 的破坏半径就不是"重试一次"能解决的了。

## 问题

社区里最常见的两个担心：一是 exec 跑在宿主机上，`cd` 到错误目录后误删；二是文件工具被诱导越权访问工作区之外的路径——提示注入、恶意链接、粘贴进来的内容都可能成为输入源。默认配置必须保证一件事：**最坏情况下，损失被圈在一个可丢弃的目录里**。

## 做法：四层约束

OpenClaw 的 sandbox 不是单点开关，而是分层设计：

1. **文件工具的路径域**。read/write/edit 以 workspace 为根解析路径，越出根目录的操作直接拒绝。这是第一道墙，拦住大部分"温和的误操作"。
2. **exec 的容器隔离**。sandbox mode 设为 `all` 后，shell 命令在独立 Docker 容器内执行。容器里只挂载 sandbox 目录——它是 workspace 的一份**拷贝**，不是软链。Agent 在里面执行 `rm -rf`，删的是这份拷贝，宿主机原目录原封不动。
3. **会话策略**。`non-main` 模式下，来自群聊或新联系人的会话默认进 sandbox；配合 DM pairing / allowlist，未配对用户根本走不到工具调用那一步。
4. **elevated 门禁**。少数确需在宿主机执行的命令走 elevated 通道，逐条确认，建议维护 allowlist 而不是随手放行。

验证方法很朴素：准备一个含哨兵文件的测试任务，诱导 Agent 执行一次破坏性删除，然后检查宿主机 workspace 是否完好、sandbox 内的拷贝是否被清空。五分钟就能确认这套模型在你的机器上真的生效。

## 踩坑点

- 图省事把全局 sandbox mode 设为 `off`，之后忘了改回来。建议 `off` 只出现在临时调试的 agent 上，用 per-agent override 而不是全局关闭。
- 往 sandbox 容器里 bind-mount 宿主机目录"方便一下"，隔离立刻失效。尤其不要挂 `~` 或任何含密钥的目录。
- 把 `docker.sock` 挂进容器或用 `--privileged`，等于交出根钥匙——容器内一条命令就能操控宿主 Docker。
- sandbox 默认不限制出网。**文件删不掉，不等于数据出不去**，敏感环境要额外配 egress 策略。
- macOS/Windows 上 Docker Desktop 与 Linux 的挂载路径、快照构建速度有差异，CI 里验证过的配置本机未必一致。
- Agent 在 sandbox 里改完文件没回同步，误以为是"它丢了我的修改"，于是去关 sandbox——先确认同步方向再动手。

## 可复用建议

- 生产环境固定 `mode: "all"`，例外走 per-agent 配置。
- workspace 进 git，sandbox 拷贝只当缓冲区，不当备份。
- gateway 只绑 loopback + token，不要暴露到公网。
- 定期用"哨兵文件 + 故意 rm"做破坏性演练，把安全配置当代码维护，写进 checklist。
- 保留 exec 审计日志，事后能回答"它到底跑了什么"。

## 总结

Agent 误删文件的防线，不靠在 prompt 里写一句"请小心"，而靠默认架构：路径域限制拦第一层，容器隔离兜底，会话策略收窄入口，elevated 门禁管住残余的高权限需求。OpenClaw 的设计哲学是**假设模型一定会犯错，然后让错误的代价可控**——这也是你敢把自动化任务交给它过夜的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/262a1e489bdd8b04.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/312424134550ad1b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/a58b9a1862556de4.png)

