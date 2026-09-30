---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39919
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

OpenClaw 的 agent 默认带 exec 和文件读写工具，能跑 shell 就意味着理论上能执行 `rm`。很多刚上手的人第一反应是："这玩意会不会把我的目录删了？"这个担心合理，但方向不对——问题从来不是"模型会不会犯错"，而是它犯错时爆炸半径有多大。OpenClaw 的 sandbox 安全模型就是围绕这一点设计的。

## 它实际做了什么

写在 system prompt 里的"不要删文件"只是软约束，真正兜底的是四层硬边界：

1. **工作区路径边界**。文件工具以 workspace 为根，路径先做 resolve（解析 symlink、拼绝对路径），再校验是否落在 workspace 前缀内。也就是说 `../` 和软链逃逸是在 resolve 之后被拦，而不是简单匹配字符串前缀。
2. **沙箱容器**。打开 sandbox mode 后，exec 类工具跑在 Docker 容器里，workspace 以 bind mount 挂入。容器内没有宿主密钥、没有 docker socket，进程失控也出不去。
3. **工具策略 deny**。tool policy 支持 deny 规则，匹配 `rm -rf`、`mkfs`、`dd of=`、`find -delete` 这类模式，deny 优先于 allow。LLM 输出再花哨，命令在网关层就进不来。
4. **执行审批**。非白名单命令走 exec approvals，需要你在对话里确认才放行。模型可以提议，人负责点头。

外加第五层可恢复性：workspace 纳入 git 或定时快照。前四层回答"能不能"，这一层兜的是"万一呢"。

## 踩坑点

- **把整个 HOME 挂进容器**。沙箱还在跑，但边界已经没了，这是社区里最常见的事故姿势。
- **只写 allow 不写 deny**。allowlist 挡不住变体：`unlink`、`find -delete`、`shutil.rmtree`。deny 规则要按"意图"列，不按"命令名"列。
- **容器里留了 sudo 或 privileged**。模型很擅长发现并使用它。
- **workspace 里放了指向外部的软链**。resolve 后越界会被拦，但如果你用 mount 把外部目录接进来，那条路径就是合法通道，要想清楚再挂。
- **只保护 workspace**。持久卷、日志目录、缓存这些"沙箱外但 agent 够得着"的路径同样会被写坏。

## 可复用建议

- 默认拒绝，显式放行。策略从"全开再堵漏"改成"全关再开洞"，维护成本反而更低。
- 每个 agent 独立 workspace，不要让两个 agent 共享可写目录。
- 高风险项目强制开启 exec approvals，宁可多点几次确认。
- 保留 exec 审计日志，出事后能还原完整时间线。
- 定期自测：故意诱导模型跑一条危险命令，观察它在哪一层被拦。这比读十遍文档有用。

## 总结

OpenClaw 防误删不靠模型自觉，靠的是路径边界 + 容器隔离 + 策略拦截 + 人工审批的分层设计，每一层都假设上一层会失败。模型一定会犯错，工程的价值在于让犯错变得便宜。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/47148a43f27ab2db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/a0f3aae62c98e12b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/4e7deeff34c08d07.png)

