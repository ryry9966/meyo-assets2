---
title: Agent 的 TOOLS.md：管理本地配置与环境差异的正确姿势
feedId: 43133
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

OpenClaw 的 agent 行为，很大程度上取决于上下文里有什么。workspace 里的 AGENTS.md 解决“这个 agent 是谁、该遵守什么约定”，但另一类信息长期没有合适的落点：**这台机器上工具的真实状态**——路径、包管理器、端口、代理、哪些命令能用哪些不能用。结果就是模型靠猜：在 Ubuntu 上猜 `brew`，在 macOS 上猜 `apt`，工具明明装了，却因为路径不对而反复失败。

`TOOLS.md` 就是给这类信息的正式落点：放在 workspace 根目录，随会话注入上下文，专门回答一个问题——**在本机怎么正确使用工具**。

## 问题

单机时这些问题还能忍，多机（笔记本 + homelab + VPS）一起跑时就集中爆发：

- 同一条指令，两台机器行为不一致，排查成本极高；
- 工具调用失败后 agent 反复试错，浪费 token，还可能留下垃圾文件；
- 这些细节写进 AGENTS.md 会把它撑爆，写进全局配置又没法按机器区分。

## 做法

**第 1 步：建文件。** 在 workspace 根目录放 `TOOLS.md`。多机场景就每个 workspace 一份——文件跟着 workspace 走，天然按机器隔离。

**第 2 步：按“事实 / 判断 / 边界”三段写：**

```markdown
## 机器事实（自动采集）
- OS: Ubuntu 24.04，用 apt，没有 brew
- node 由 nvm 管理，调用前先 source
- gateway 端口 18789，代理 127.0.0.1:7890

## 使用判断（人工维护）
- 截图用 /usr/local/bin/shot，不要自己拼 imagemagick
- python 一律走 uv，别碰系统 python

## 边界（禁止项）
- 不要 sudo
- 持久化只写 ~/data
```

**第 3 步：事实靠脚本采集。** 写十几行 shell，输出 `uname`、`which`、关键版本号，贴进“机器事实”段再手工补注释。别手抄版本号——手抄必错。

**第 4 步：定期验证。** 每月让 agent 做一次环境自检：按 TOOLS.md 逐条执行，报告不一致的条目。文档和现实的 diff，比 git diff 更值得看。

## 踩坑点

- **写成散文。** 长段落模型提取不可靠，短条目 + 代码块最稳。
- **留过期信息。** TOOLS.md 里写错的路径比不写更糟——模型会优先信它。暂时删不了就标注“未验证”。
- **写秘密。** API key、token 一律不进这个文件，只写“从哪个环境变量读”。
- **贪多。** 整个文件都会占上下文预算，控制在几十行；长细节拆成子文档，按需引用。
- **边界太模糊。** “小心操作”没有约束力，“不要 sudo、不要动 /etc”才有。

## 可复用建议

- **分层：** 跨机器的约定放 AGENTS.md，机器相关的事实放 TOOLS.md，两者别混。
- **入 git（脱敏后）：** 机器之间的 diff 能直接暴露环境漂移，比事后排查便宜得多。
- **模板化：** 把“采集脚本 + 三段模板”存成一个小 repo，新机器十分钟就能产出一份合格的 TOOLS.md。

## 总结

TOOLS.md 的本质，是把本机环境的 ground truth 固化成 agent 可读的上下文。原则就四条：**短、准、无秘密、有验证**。建议从你失败率最高的那台机器写起，一次只补真实踩过的坑——这份文件才会越用越准。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/5b999e3b3dba7244.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/0219f792dd9d1f94.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/d493798284ba1786.png)

