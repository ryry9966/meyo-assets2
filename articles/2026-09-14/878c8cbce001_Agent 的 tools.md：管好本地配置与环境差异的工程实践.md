---
title: Agent 的 tools.md：管好本地配置与环境差异的工程实践
feedId: 37440
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

OpenClaw 这类常驻 agent 的实际能力，大部分落在本机工具上：shell、脚本、各种 CLI。而 agent 每次会话都是"新来的"，对这台机器一无所知。workspace 里的 tools.md 本质上是一份写给 agent 的环境说明书：装了什么、装在哪、有什么本机特有的限制。

现实是，很多人的 tools.md 要么只有两三行，要么塞满了和工具无关的内容，基本形同虚设。

## 问题

缺一份维护良好的 tools.md，常见症状有四类：

- **反复探测环境**：每次会话都跑 `which node`、`ffmpeg -version`，白白消耗 token 和轮次。
- **路径错配**：nvm 装的 node 在非交互 shell 里找不到；Apple Silicon 上 homebrew 在 `/opt/homebrew/bin`，agent 却按老经验找 `/usr/local`。
- **跨机踩雷**：同一份 workspace，笔记本和服务器行为不同——服务器上 docker 要 sudo、出网走代理、没有 GPU。
- **秘密混入**：把 API key 直接写进 tools.md 图省事，结果跟着同步、备份、截图到处泄漏。

根因一句话：这台机器的"常识"只存在你脑子里，agent 没有持久常识，只能每次暴力试错。

## 做法

核心原则：**只写 agent 无法通过一次成功执行推断出来的事实，且每条尽量可验证。**

```markdown
# Tools

## 运行时
- node: nvm 管理，路径 ~/.nvm/versions/node/v20.x/bin
  非交互 shell 需 source ~/.nvm/nvm.sh，或用绝对路径
- python: 系统 3.12，勿动

## CLI
- ffmpeg: /opt/homebrew/bin/ffmpeg，含 libx264
- docker: 需 sudo，daemon 不常驻，先 docker info 探活

## 网络
- 出网走 http://127.0.0.1:7890 代理
- 内网 GitLab 仅办公网可达

## 限制
- 无 GPU；长任务用 nohup
- 磁盘紧张，下载产物统一放 /tmp
```

维护流程：

1. **盘点**：从 agent 的执行记录里找出它真在用的工具，而不是你希望它用的。
2. **记事实**：路径、版本、前置条件、失败特征，并标注"上次验证日期"。
3. **差异分档**：机器级差异（路径、代理）进 tools.md；行为偏好进 AGENTS.md；进度状态进 memory，三者不要混。
4. **对账**：agent 每次因环境报错，修完后把结论补回 tools.md，让它成为排障结论的沉淀层。

## 踩坑点

- **写成教程**："ffmpeg 怎么转码"不该出现，只写"本机这版有什么不同"。
- **写死易变项**：IP、端口、版本号会漂移，过期条目宁可删。
- **塞秘密**：token 一律走环境变量，tools.md 只写"凭据在哪个变量里"。
- **多机共用一份**：环境不同就拆成 tools-desktop.md / tools-server.md，入口文件写清按 hostname 选用。
- **过长**：几百行会随每次会话注入，token 成本和信噪比双输；控制在几十行，冷门信息放子文件按需读取。

## 可复用建议

把 tools.md 当成"机器的 /etc/hosts"：短、准、只放必需映射。每条带验证命令，让 agent 能自查条目是否仍成立。多机场景用模板加差异覆盖，不要各写各的。排障后五分钟内更新文档，隔天就忘。

## 总结

tools.md 的价值不在"写了"，而在持续准确反映这台机器的真实状态。只写 agent 推断不出的环境事实、把差异显式化、把秘密挡在外面、定期对账——做到这四点，agent 在任何一台新机器上的第一分钟，就能像熟手一样干活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/b51b33094bdd9f7a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/6dfbbb206daafb0f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/ebb845d63ca0bb46.png)

