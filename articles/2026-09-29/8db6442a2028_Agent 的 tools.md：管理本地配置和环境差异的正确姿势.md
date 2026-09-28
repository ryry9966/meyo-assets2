---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39455
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

跑本地 Agent 最烦的不是写 prompt，而是环境。同一套 skill，在家里 Linux 主机上跑得好好的，换到公司 MacBook 就开始报错：python 路径不对、代理没走、ffmpeg 找不到。这些信息模型本身不知道，全靠它"猜"，猜错了就浪费一轮工具调用。

OpenClaw 的 workspace 机制给了我们一个天然挂载点：agent 每次会话都会读取工作区里的说明文件。`tools.md` 就是其中最值得认真维护的一个——把"这台机器长什么样"写成数据，而不是散落在 prompt 和口口相传里。

## 问题

实际项目里常见的三种坏味道：

1. **硬编码在系统提示词里**：换机器、换路径就得改 prompt，改完还容易漏。
2. **信息散落各处**：MCP 配置一个文件、插件参数一个文件、shell 别名又一个文件，agent 没有统一入口能"看懂"环境。
3. **凭记忆口述**：每次对话现场告诉 agent"python 在 conda 里"，说一次忘一次。

## 做法

我的实践分四步：

**1. 建 base + local 两层。** 仓库里放 `tools.md`（通用基线，进 git），机器差异放 `tools.local.md`（不进 git）。读取顺序固定：base → local，后者覆盖前者。

**2. base 只写结构性事实**，按固定小节组织：

- 运行时：python / node 版本、包管理器
- 路径：工作区、venv、数据目录
- 网络：代理、镜像源（只写变量名如 `$HTTPS_PROXY`，不写值）
- MCP：server 名字、传输方式、启动命令
- 平台差异：macOS 用 brew、Windows 注意编码之类

**3. local 只写覆盖项**，比如"这台机器 ffmpeg 在 /opt/homebrew/bin"。文件顶部留一行"上次核验日期"，方便判断新旧。

**4. 让 agent 参与维护。** 系统提示词里加一条规则：执行命令前先查 tools.md；发现与文档不符的事实（工具不在预期路径等），修正 local 文件而不是硬扛。配合 git diff，人只做 review。

## 踩坑点

- **别把密钥写进去**。写环境变量名和获取方式，值一律走 secret manager。tools.md 进 git，就当它是公开文件。
- **别当垃圾桶**。超过 200 行说明该删了。过期信息比没有信息更糟——agent 会拿旧事实反复撞墙。
- **别和 MCP 配置重复声明**。tools.md 负责"描述环境"，MCP config 负责"声明能力"，写重了必然漂移。
- **别假设 agent 会自觉**。文档里加一句"命令失败时先核对 tools.md 再重试"，比指望它主动可靠得多。

## 可复用建议

- 把 base 做成 repo 模板：新机器初始化只要 clone → 复制 local 模板 → 填覆盖项 → 让 agent 跑一轮探测自动补全，十分钟搞定。
- 定期（或 CI 里）跑一个 env-doctor 脚本，核对文档里的路径、版本是否仍然成立，把"文档腐化"变成可检测问题。
- 团队只共享 base；local 天然隔离了个人目录、公司代理这类敏感差异。

## 总结

tools.md 的本质是把环境当作数据管理：进 git、分两层、可核验、让 agent 一起维护。成本很低，但能把"换台机器重调半天"压到几分钟。建议从一个最小模板起步，跑两周、按真实失败案例往里补——这类文档是长出来的，不是一次写完的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/a6a8547e8bc94317.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/996f7259ef13db27.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/81bc0b22d7f3e43d.png)

