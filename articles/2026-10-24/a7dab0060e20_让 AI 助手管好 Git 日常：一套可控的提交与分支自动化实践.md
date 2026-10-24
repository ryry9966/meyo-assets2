---
title: 让 AI 助手管好 Git 日常：一套可控的提交与分支自动化实践
feedId: 42450
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

日常开发里大量时间花在机械操作上：写 commit message、切分支、删已合并分支、整理暂存区。OpenClaw 的 Agent 有 shell 和 MCP 能力之后，这些活理论上都能交出去。但直接把 git 权限裸给模型，第一周就会出事。这篇记录我这套跑了两个多月的配置：哪些交出去，哪些锁死。

## 问题

裸跑有三个典型翻车现场：

1. 一次性 `git add .` 加一句模糊 message，把调试代码和功能改动混进同一个提交；
2. 遇到冲突时自作主张 `reset --hard`，丢掉本地未推送的改动；
3. 分支命名随心所欲，已合并分支越堆越多。

根因是模型对"安全边界"没有直觉，prompt 里写一句"小心点"没有任何约束力。

## 做法

核心原则：**白名单放行、黑名单硬拒、写操作前先出计划。**

**1. 工具层收权。** Agent 走 MCP git server 或 shell 时，只暴露 `status / diff / log / add / commit / branch / checkout` 这类只读或低危命令。危险操作在包装脚本里拦截，而不是靠 prompt 祈祷：

```bash
#!/usr/bin/env bash
# ~/bin/git-agent-wrapper
DENY='reset --hard|push --force|push -f|clean -fd|checkout -- \.'
if echo "$@" | grep -qE "$DENY"; then
  echo "[blocked] needs human approval" && exit 1
fi
exec git "$@"
```

Agent 的 git 命令统一指向这个 wrapper。

**2. 约定落盘。** 在仓库根放一个 `AGENTS.md`：conventional commits、分支命名 `feature/xxx` `fix/xxx`、单提交单关注点。让 Agent 每次会话先读它。约定跟着仓库走，不塞全局 prompt。

**3. 计划先行。** 提交前 Agent 必须先输出三样东西：涉及文件、分组建议、拟用 message。人回一个 "ok" 才执行。分支清理同理：先跑 `git branch --merged main` 给出候选删除清单，确认后再删。

**4. 提交前扫描。** 链路里加一步 gitleaks，命中密钥直接中止。这一步不要交给模型判断。

## 踩坑点

- **cwd 漂移**：多仓库环境下 Agent 偶尔在错误目录执行 git。解法是 wrapper 里校验 `pwd` 必须在登记的仓库列表内。
- **add 粒度**：即便有约定，模型仍爱 `git add .`。后来在 wrapper 里对 add 限制为必须带显式路径，才真正按文件分组提交。
- **message 过度发挥**：不加约束会写出营销腔。给两个 few-shot 样例、限定 `type(scope):` 格式后稳定很多。
- **审计缺失**：起初没记日志，出问题无法回溯。现在每次 wrapper 调用都追加到本地 log，含时间、命令、退出码。

## 可复用建议

- 危险操作用代码拦截，别用 prompt 拦截；
- 所有写操作保持 plan → confirm → execute 三段式，哪怕多花十秒；
- 约定写进仓库文件而非全局配置，人和 Agent 共用一套真相；
- `push` 始终留给人，Agent 最多把命令准备好放在那。

## 总结

两个月下来，Agent 承担了约八成的提交整理和分支清理，message 质量比我手写更稳定，事故零起——不是因为模型聪明，而是因为危险路径全被堵死了。自动化 Git 的正确姿势不是"让 AI 更像工程师"，而是给它一个结构上就不允许犯大错的工位。整套配置半小时能搭完，欢迎在评论区贴出你的 denylist，互相抄作业。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/35a4844b51a7a9be.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/4327796033f567b2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/4c9be3c6eb726fa4.png)

