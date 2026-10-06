---
title: Agent 的 TOOLS.md：把它当成本机差异清单来维护，而不是备忘录
feedId: 40744
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 的 agent 在每次会话启动时，会把 workspace 下的 TOOLS.md 注入系统提示。很多人把它当成可有可无的备注文件，但它其实是 agent 唯一稳定的“本机说明书”：仓库里有 `.env` 和 config 管应用配置，可 agent 自己要跑的 shell、CLI、浏览器 profile、本地服务端口，这些环境事实只存在于你的机器上，不存在于任何代码里。

## 问题

单机时矛盾不显，环境一多就暴露：

- 换一台机器，agent 凭训练记忆猜路径：ffmpeg 不在 PATH、`python` 和 `python3` 混用、`docker compose` 子命令写法不同；
- 多机同步 workspace 时，A 机的端口和 profile 信息污染 B 机；
- 修一次错一次：agent 上周报过“命令不存在”，这周继续踩同一个坑，因为没人把修正写回上下文。

## 做法

我把 TOOLS.md 当成“本机差异清单”来维护，四个规则：

1. **只写事实，不写行为。** 行为规范放 AGENTS.md；TOOLS.md 里是常用 CLI 的绝对路径、版本、本地服务端口、浏览器 profile 名。每条一行，条目式。
2. **密钥只写位置，不写值。** 写 `token 在 ~/.config/foo/token，读取用 cat`，绝不把 key 本身贴进去——TOOLS.md 会被注入每个会话，等于明文广播。
3. **本机文件不入库。** gitignore 掉 TOOLS.md，仓库里只留 `TOOLS.md.example` 模板；或者写一个 20 行的 setup 脚本，用 `which`/`uname` 检测环境后自动渲染，换机器跑一遍即可。
4. **踩坑回写。** agent 报“命令不存在”“路径不对”时，当场把正确事实追加进去；顺手删失效条目。每条带最后验证日期，过期就删，防止上下文膨胀。

## 踩坑点

- **改了不生效。** 注入发生在会话启动时，改完记得 `/new` 开新会话，否则 agent 读到的还是旧版本。
- **写成长文。** 有人把它写成散文，agent 反而抓不住重点。控制在 100 行以内，短条目优先。
- **平台差异不标注。** macOS 和 Linux 的命令、路径写法不同，条目前缀标清机器名或平台。
- **和 AGENTS.md 重复。** 同一工具两边各写一份，改一处忘一处，最后 agent 拿到的是互相矛盾的指令。

## 可复用建议

一个最小模板，直接抄走：

```markdown
# TOOLS.md (workstation-a, macOS, 更新 2025-06-12)
## CLI
- ffmpeg: /opt/homebrew/bin/ffmpeg (7.1)
- node: v22 via nvm，先 `source ~/.nvm/nvm.sh`
## 本地服务
- dev api: 127.0.0.1:8787（先 `make up`）
## 凭证
- gh cli 已登录，不要读文件里的 token
```

原则一句话：AGENTS.md 管它该做什么，TOOLS.md 管它脚下的地面长什么样。

## 总结

TOOLS.md 的正确姿势是“给 agent 的本机 README”：管差异、管事实、条目化、密钥只给位置、配合模板和生成脚本。做到这几点，多机环境下的 agent 行为才谈得上稳定复现，而不是每台机器重新调教一遍。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/e66172cf8785bba0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/04de5654e2414e7f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/6e37be50b441cd7e.png)

