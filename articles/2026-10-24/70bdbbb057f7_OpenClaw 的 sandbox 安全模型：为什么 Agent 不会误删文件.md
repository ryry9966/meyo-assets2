---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42426
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

OpenClaw 的 agent 默认带 exec / file 工具，能跑命令、读写文件。这类能力一旦失控，最怕的不是答错问题，而是把 `rm -rf` 落在错误的目录上。社区里见过不止一次「让 agent 清理临时文件，结果把整个项目目录清了」的事故。OpenClaw 给出的答案不是「把提示词写得更小心」，而是一套默认开启的纵深防御。

## 问题

误删的本质是：**agent 的计划没错，但作用域错了**。常见三种根因：

1. **工作目录漂移**——agent 在错误的 cwd 下执行相对路径命令；
2. **通配符过宽**——`*.log` 匹配到了不该匹配的文件；
3. **指令歧义**——「清理一下」被理解成「删掉一切可删的东西」。

提示词层解决不了这些，因为 LLM 的失败是概率性的。防线必须落在执行层。

## 做法：四层防线

1. **沙箱执行（默认值）**：exec 工具默认在 Docker 容器内运行，文件系统是容器内部的。agent 真执行了 `rm -rf /`，删掉的也只是容器层，随容器销毁。
2. **挂载收敛**：只有 workspace 被 bind-mount 进容器，且默认只挂业务需要的路径。挂载是白名单，不是「把整个 home 给它」。
3. **工具策略**：gateway 的 tool policy 按 agent 粒度做 allowlist。跑批 agent 可以有 exec，但 deny 掉 workspace 外的写入；浏览型 agent 干脆不给 exec。

   ```yaml
   # 示意，字段以你部署的版本为准
   agents:
     cleaner:
       sandbox: docker
       mounts: [/srv/jobs/tmp]
       tools: { allow: [exec, read], deny: [write] }
       approvals: destructive
   ```

4. **审批门**：elevated / ask 模式下，破坏性命令（`rm`、`chmod`、覆盖式 `mv` 等）需要人工确认，审批记录进日志，可回放。

四层叠加后，「误删」被压缩成一个可丢弃的容器内事件。

## 踩坑点

- **图省事关掉 sandbox**：容器冷启动几秒确实烦，但它是整个模型的地基，关掉等于裸奔。
- **挂载了整个 home**：沙箱被直接架空，容器里删的就是宿主机文件。
- **审批疲劳**：全量 ask 模式会让人盲点「同意」。要按风险分级，只对破坏性操作弹窗。
- **MCP 文件服务越权**：MCP server 自己配置的根路径不受 tool policy 管，需要在 server 端再限一次。
- **软链接逃逸**：workspace 里放了指向家目录的 symlink，容器内读写会穿出去。挂载前清一遍。

## 可复用建议

- 把 agent 当成一个**权限受限的实习生**：能读什么、能写哪里、能跑什么，全部显式配置，不靠信任。
- 每个 agent 给最小工具集，需要时临时加权限，用完收回。
- workspace 保持小而专；一次性任务用独立目录，不要混入真数据。
- 保留执行日志和快照：出事能回放，也能复盘整条审批链。
- 定期做「故意诱导」测试：让 agent 执行一条边界命令，验证防线真的在，而不是只在文档里。

## 总结

Agent 误删不是概率问题，是架构问题。OpenClaw 的思路是把「不做坏事」从模型责任转移为系统约束：默认沙箱、最小挂载、白名单工具、分级审批。提示词依然重要，但它只是最外面那道软防线——真正的安全感，来自即使模型犯错，损失也被限制在一个可以整包丢弃的容器里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/6305591bc86fc06d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/2a80edf53a449e35.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/9b0a0753fb807d37.png)

