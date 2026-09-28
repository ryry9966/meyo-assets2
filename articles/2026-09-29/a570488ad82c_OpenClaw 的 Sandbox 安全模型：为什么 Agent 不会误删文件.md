---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39423
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

跑过 Agent 的人大多做过同一个噩梦：让它清理临时目录，它顺手把整个项目 `rm -rf` 了。OpenClaw 是常驻本机的个人助理网关，agent 手里同时握着文件读写工具和 exec 执行工具，这个风险不是假设，是日常。所以值得把它的 sandbox 安全模型拆开看一遍。结论先说：agent 不误删文件，靠的不是模型"聪明"，而是分层约束。

## 问题

常见做法有两类：一是全靠 prompt 约束（"请不要删除文件"），二是干脆不给执行权限。前者是祈祷，后者废掉 agent 一半能力。OpenClaw 走了第三条路：给能力，但把能力关进沙箱。

## 机制：四层防线

**1. 工作区作用域（workspace scoping）**
文件读写类工具（read/edit/write 这类）的路径会先被规范化，再约束在 workspace 根目录内。`../../etc/passwd` 式的相对路径逃逸、以及指向宿主目录的 symlink，都在路径解析阶段被拦下。这是第一道静态闸门。

**2. exec 的 Docker 沙箱**
shell 命令不走宿主，而是丢进容器：workspace 以 bind mount 挂进去，容器内看不到你的家目录、SSH key、crontab，网络出口可以单独限制。即使模型抽风生成了 `rm -rf ~`，容器里那个 `~` 也不是你的 `~`。

**3. 工具策略与提权审批**
tool policy 可以按 allow/deny 收紧单个工具；危险命令（sudo、全局写、装包）走提权确认流程，需要人在网关侧点头。策略写在配置里，不写在小作文里。

**4. 会话分级**
默认 `sandbox.mode: non-main`——群组/渠道触发的会话在沙箱里跑，主会话才有宿主权限（具体键名以你部署版本的文档为准）。日常试险应该去群组会话，而不是主会话。

## 踩坑点

- **"文件不存在" ≠ 文件没了**：沙箱里只挂载了 workspace。agent 说找不到 `/data/xxx`，多半是没挂载，别让它去"修复"。
- **symlink 逃逸**：workspace 里一个软链指向宿主目录，等于自己开了个洞。挂载前先检查。
- **root 写出的文件**：容器内以 root 运行时，写出的文件属主是 root，宿主上清理很麻烦，注意统一 UID。
- **把 docker socket 或无限制 sudo 交给 agent**：等于亲手拆掉沙箱地基，上层所有防线同时作废。
- **主会话错觉**：默认配置下主会话不在沙箱里，危险实验别在这里做。

## 可复用建议

1. **分级存储**：不可再生数据放 workspace 外，需要引用就只读挂载。
2. **收紧靠配置不靠 prompt**：tool policy 和 sandbox mode 是确定性的，模型遵循度是概率性的。
3. **workspace 用 git 管理 + 定时快照**：最后防线永远是备份，不是沙箱。
4. **先在群组会话试跑一周**，开审计日志，看它实际调用了哪些工具、碰了哪些路径。

## 总结

OpenClaw 的思路是纵深防御：路径解析挡越权、容器挡误操作、策略挡滥用、审批挡意外。模型犯错是常态，架构兜底才是工程。与其反复叮嘱 agent "别删文件"，不如让它根本删不到。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/d68a6a4c2e6736c8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/7b42db43890d17a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/c1b6da980460349a.png)

