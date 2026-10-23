---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42410
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

用 Agent（OpenClaw、各类 CLI Agent、MCP 客户端）跑本地自动化一段时间后，你会发现它能不能干活，往往不取决于模型多聪明，而取决于它对这台机器了解多少。工具路径、版本、代理、端口占用、GPU 有无——这些环境事实平时散落在 shell rc、.env、README 和聊天记录里，Agent 每次都在重新猜。

tools.md 的思路很简单：在固定位置放一份 Markdown，把 Agent 需要的环境事实显式写下来，让它随上下文加载。在 OpenClaw 里，workspace 自带的 TOOLS.md 就是干这个的，由 Agent 自己读、自己维护。

## 问题

三个典型场景：

- 同一套配置，MacBook 和 Linux 服务器表现不一致：前者 ffmpeg 在 `/opt/homebrew/bin`，后者在 `/usr/bin`，Agent 猜错就报错。
- 团队仓库里写死“用 pnpm”，对你是共识，对新同事的机器不是，Agent 按文档执行直接失败。
- 上周告诉过 Agent“这台机出网要走本地代理”，它记在会话里，这周开新会话全忘了。

根因是环境知识没有显式化、没有分层，全靠会话记忆和口头约定。

## 做法与步骤

**1. 分两层，职责分开**

- 全局层：workspace 的 `TOOLS.md`（如 `~/.openclaw/workspace/TOOLS.md`），只写“这台机器”的事实：工具链及路径、代理、GPU/CUDA、已知坑。不进项目仓库。
- 项目层：仓库根目录的 AGENTS.md 或同等文件，进版本控制，只写“这个项目”的约定：包管理器、启动命令、禁止触碰的目录。

**2. 写可验证的事实，不写散文**，每条尽量带验证方式：

```markdown
## 工具链
- node: v22（nvm 管理），验证 `node -v`
- ffmpeg: /opt/homebrew/bin/ffmpeg（brew）

## 已知坑
- 8080 被常驻服务占用，dev server 用 8081
- 出网需代理 http://127.0.0.1:7890

## 禁止
- 不要改 /etc/hosts
```

**3. 控制篇幅**。tools.md 会进上下文，一屏以内最好，只放 Agent 真会用到的。

**4. 建立更新闭环**。Agent 执行中发现新事实（命令不存在、端口冲突、权限问题），当场补进 tools.md。可以在 AGENTS.md 里明文要求它这么做，把 tools.md 当环境记忆，而不是一次性文档。

## 踩坑点

- **写成小作文**：段落式的背景介绍对 Agent 没有增量，白耗上下文。
- **写入密钥**：tools.md 会被读进模型上下文，任何 key/token 只能写变量名引用，绝不写值。
- **记录运行时可探测的东西**：`which ffmpeg` 一秒能查到的事，写死反而会过期。原则：能探测的不写，写下来的必须准。
- **版本号写死到补丁位**：写"node v22 + 验证命令"，比写"v22.11.0"存活期长得多。
- **假设框架会自动读**：不同 Agent harness 对这类文件的发现机制不同，先确认你的环境真的加载了它，别自我感动。

## 可复用建议

- 新机器初始化时，把写 tools.md 当作装机清单的一部分：装完环境顺手核一遍，相当于给自己的 runbook。
- 项目层文件可以反哺 onboarding：新人读它就知道环境约定，一份文档两用。
- 团队已有 setup 脚本的话，让 tools.md 与脚本互相引用，保持单一事实源，避免两处漂移。

## 总结

tools.md 不是新协议，就是几 KB 的 Markdown 约定。它的价值在于把“Agent 与这台机器的契约”从隐式变成显式、从会话记忆变成落盘状态。分层写、写事实、能探测的不写、发现即更新——做到这四条，Agent 在不同环境下行为的一致性会有肉眼可见的提升。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/b9bc2483b8e6186e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/36af8e441e7e8e42.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/981532acf18e2d09.png)

