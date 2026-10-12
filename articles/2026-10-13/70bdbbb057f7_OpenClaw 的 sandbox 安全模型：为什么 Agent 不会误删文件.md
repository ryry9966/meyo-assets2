---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41349
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

OpenClaw 的 Agent 能力来自工具调用：exec、文件读写、浏览器控制、各类 MCP 与插件。这些工具一旦被模型"想歪"，最坏结果不是答错一句话，而是对真实文件系统做不可逆操作。所以 OpenClaw 的设计假设很保守：默认所有 LLM 输出都不可信，真正的安全边界在工具层和执行层，而不是模型层。

## 问题

误删文件的典型路径不是"模型故意作恶"，而是三种更常见的工程失误：

1. **路径解析错误**——相对路径、`~` 展开、工作目录漂移，`rm -rf ./build` 实际落在了别的目录；
2. **上下文污染**——长会话中早期指令被压缩或截断，模型依据残缺"记忆"执行清理；
3. **插件越权**——某个 MCP server 拿到了接近宿主机的文件权限，Agent 一次常规调用被放大成全局破坏。

## 做法：四层防线

OpenClaw 的 sandbox 模型是典型的纵深防御，任何一层失守都不直接等于事故：

1. **工作区隔离**。文件工具默认 chroot 到 workspace，路径参数先做规范化（resolve + realpath），越界路径在工具层直接拒绝，根本不会下发到 shell。
2. **容器沙箱**。exec 类工具运行在 Docker 容器内，只挂载 workspace，容器内以非 root 用户运行。即使命令被拼接出灾难性内容，作用域也被锁死。
3. **工具权限矩阵**。每个 Agent profile 声明可用工具集合；写操作和 exec 默认需要 approve，读操作放宽。递归删除、大面积通配符匹配等危险模式会被 gateway 策略层拦截，要求显式确认。
4. **会话级凭证与审计**。插件/MCP 按会话注入最小凭证，所有工具调用落审计日志，出问题可以回放到具体某一次调用。

## 踩坑点

- 图省事把宿主机 `/` 或整个 HOME 挂进容器，第二层防线直接失效；
- 容器内用 root 跑，除了 mount 后文件属主混乱，越界写入也更容易得手；
- 为了"全自动"把 approve 全关，等于只剩路径校验一层在兜底；
- 符号链接逃逸：workspace 里一个指向外部的 symlink 就能让 realpath 校验形同虚设，重要目录建议只读挂载；
- 某些 MCP server 自带 shell 能力，等于绕过了工具矩阵，接入前必须审一遍它的工具清单。

## 可复用建议

- 新接入任何插件/MCP，先在一次性容器里跑通，再决定是否授予持久 workspace；
- 定期做"破坏性演练"：故意让 Agent 执行 `rm`，观察是哪一层拦截、审计日志是否完整；
- approve 策略按工具粒度配置，而不是用全局开关；
- 密钥、生产配置等敏感目录，物理上不要出现在 workspace 可达路径里。

## 总结

Agent 不误删文件，靠的不是模型更聪明，而是假设它一定会犯错之后的工程兜底。OpenClaw 把安全性放在工具层和执行层，模型层只负责"提案"。这个思路可以直接迁移：任何让 LLM 碰真实系统的项目，都值得先画一遍这四层防线，再决定哪里可以放宽。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/bb29fdd2f339fdea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/41ba1c5abb597549.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/c6a7a91e23783bf7.png)

