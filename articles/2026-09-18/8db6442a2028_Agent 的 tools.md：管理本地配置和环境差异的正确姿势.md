---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 38035
source: 综合讨论
publishedAt: 2026-09-18
---

# 背景

OpenClaw 的 Agent 上手干活前，要先回答一个朴素的问题：它到底跑在什么样的机器上。Linux 还是 macOS、brew 还是 apt、Node 由 nvm 还是 fnm 管、Python 要不要先激活 venv、项目在 `~/code` 还是 `/srv`——这些环境事实 Agent 默认一概不知。MCP 和插件提供的是能力，而"这台机器长什么样"，目前最省事的载体仍然是 Agent 工作区里的一份 Markdown，也就是 tools.md。

# 问题

不写这份文件，典型症状是：

- Agent 在 Ubuntu 上给你 `brew install`，在 mac 上给你 `apt`；
- 路径张口就来 `/home/xxx`，而你实际在 `/Users/xxx`；
- 忘了激活 venv，pip 直接装进全局；
- 换台机器、换个容器，同一套指令全失效，你在对话里反复纠正。

纠错成本才是大头：每次会话都重新解释一遍环境，等于你在给 Agent 当人肉 config。而随手写一份之后，新问题又来了——环境会变，文档会烂。Agent 对文档是全信的，一份过期的 tools.md 比没有更危险。

# 做法

核心思路四条：**分层、写事实、脚本生成、定期刷新**。

**1. 分层：三个文件各管各的**

- `tools.base.md`：跨机器通用约定，进版本库，如"优先用 pnpm"；
- `tools.<host>.md`：机器档案，脚本自动生成，不入公共库；
- `tools.<project>.md`：项目覆盖层，只写与全局冲突的部分。

加载顺序 base → host → project，后者覆盖前者。

**2. 写事实，不写散文**

读者是 Agent，不是人。它要的是能直接照做的一行行事实：

```markdown
## 环境
- OS: Ubuntu 24.04
- Node: v22（fnm 管理）
- Python: 3.12，venv 在 ~/code/proj/.venv，激活：source .venv/bin/activate

## 约定
- 装包用 apt，禁用 brew/snap
- PATH 变更写 ~/.profile，不要动 ~/.bashrc
```

**3. 用脚本生成机器档案**

手写必漏。二十行探测脚本输出 markdown，挂到 shell 初始化或 make target 上：

```bash
cat <<EOF
## 环境（$(date +%F) 生成）
- OS: $(uname -s)
- 包管理器: $(command -v brew >/dev/null && echo brew || echo apt)
- Node: $(node -v 2>/dev/null || echo none)
- Python: $(python3 -V 2>/dev/null || echo none)
EOF
```

**4. 更新策略**

环境大改后手动刷新一次；生成结果过一遍 git diff 再提交，把 tools.md 的变更当代码变更对待。base 层基本不动，host 层高频小改，project 层跟随项目演进。

# 踩坑点

1. **过期文档比没有更糟。** 文档写 Python 3.10、实际 3.12，Agent 会照着错的做。文件里保留生成日期，提醒自己刷新。
2. **别写密钥。** tools.md 会进上下文、可能进日志。只写变量名（如 `API_KEY 见 .env`），不写值。
3. **别和 AGENT.md 抢职责。** AGENT.md 写行为规范（怎么做事），tools.md 写环境事实（机器长什么样）。内容重复必然漂移。
4. **验证加载链路。** 写完问一句"这台机器用什么包管理器"，答错就是没被读入，先修位置再谈其他。
5. **Windows 混用**时在 host 档案里明确路径分隔符和换行风格。

# 可复用建议

- 命名固定为 `tools.base.md` / `tools.<host>.md` / `tools.<project>.md`，团队统一，脚本才好写；
- 生成脚本 + git diff + 定期刷新，三件套齐了文档才不会烂；
- 提交前自查：无密钥、每条可执行、有生成日期、与 AGENT.md 无重复、加载已验证。

# 总结

Agent 不会读心，环境差异是它出错的最大来源之一。tools.md 的价值不在"写文档"这个动作，而在把它当配置管理：分层、生成、版本化、定期校验。做到这四点，同一套 Agent 在笔记本、服务器、容器里表现一致，你也就不用再当人肉 config 了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/e9765a16c08ce0cb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/65e60b0a73d36b4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/7a07b5b9772a75fa.png)

