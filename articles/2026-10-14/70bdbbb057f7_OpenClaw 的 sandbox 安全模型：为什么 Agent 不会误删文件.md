---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41480
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

让 Agent 拿到 shell 和文件工具，等于把一把能递归的钥匙交给一个"概率性"的执行者。OpenClaw 的 Agent 默认具备读写、执行类工具能力，最初的担心很直接：模型幻觉出一个路径，或者被读入的文档内容里藏了一句"清理一下这个目录"，`rm -rf` 就会落在你不想删的东西上。

## 问题拆解

误删本质上不是"模型不够聪明"，而是执行面缺少边界。常见的漏洞路径有三类：

1. **路径逃逸**：Agent 生成 `../../` 或绝对路径，落到 workspace 之外；
2. **命令绕过**：用 `find -delete`、`sh -c`、管道拼接等形态绕过表面上的 rm 黑名单；
3. **权限放大**：Agent 进程跑在你的登录用户下，宿主目录一挂载，爆炸半径就是整个家目录。

## OpenClaw 的做法

OpenClaw 的 sandbox 不是单点过滤，而是分层收口：

- **路径归一化**：所有文件工具的路径先做 resolve + 规范化，符号链接解析到真实路径后再与 workspace root 比对，`..` 和软链逃逸在这一步被拦下，而不是靠字符串前缀判断；
- **能力分级**：工具声明 read / write / execute 三个 scope，写操作只允许 workspace 内，workspace 外读需显式授权，执行类命令走受限 shell；
- **命令拦截而非黑名单**：shell 工具经一个包装器解析命令结构，识别删除、移动、覆盖类操作的目标路径，逐一过策略，匹配不到白名单模式的一律转人工确认；
- **删除不落地**："删除"实际是移动到 workspace 内的 `.trash/`，带保留窗口，Agent 本身没有直接 unlink 的能力；
- **审计与快照**：每次写操作记录 diff 与调用链；会话开始对 workspace 打 git 快照，回滚是最后兜底。

## 踩坑点

- 只做字符串前缀校验，会被 `/workspace/../etc`、双斜杠、大小写变体绕过，必须用 resolve 后的真实路径比对；
- workspace 里一个指向宿主的软链，会让前面的检查全部失效，软链要在解析之后再判定；
- 把整个家目录挂进沙箱，再严的策略也只是"记录"你删了什么——爆炸半径由挂载决定，不由策略决定；
- prompt 注入可以来自被读取的文件内容，所以"Agent 说要删"和"策略允许删"必须是两条独立通道，前者只是提议。

## 可复用建议

- 默认拒绝、按需放行，工具 allowlist 比 denylist 可靠；
- 给 Agent 独立的低权限系统用户或容器，不要复用你的登录身份；
- 删除进回收站加保留期，比确认弹窗更抗手滑；
- 高危操作先 plan / dry-run，人工看过 diff 再放行；
- 每次 session 前打快照，让恢复成本低于信任成本。

## 总结

OpenClaw 能做到"Agent 不会误删文件"，不是因为模型更守规矩，而是删除这个动作在架构上就很难直达宿主文件系统：路径解析收口、能力分级、命令拦截、回收站兜底、快照回滚。策略层挡住 99% 的情况，OS 层和快照负责剩下的 1%。给 Agent 自由之前，先给它画一个跑不出去的圈子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/96d6679d1c04ae7d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/731035cb737cd175.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/5baf7deadfdd8568.png)

