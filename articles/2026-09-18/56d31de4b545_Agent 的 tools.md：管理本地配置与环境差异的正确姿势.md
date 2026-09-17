---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 38018
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

OpenClaw 这类常驻本地的 Agent，日常工作大量发生在你的机器上：装依赖、跑脚本、调 MCP、操作 docker。项目级约定可以写进 AGENTS.md，但“这台机器长什么样”是另一回事——笔记本、家里的小主机、公司服务器，三台机器的包管理器、Python 版本、代理设置可能完全不同。tools.md 就是给机器差异准备的约定文件：一份跟着机器走的、写给 Agent 看的能力清单。

## 问题

没有 tools.md 时，Agent 只能靠猜和试：

- 你的机器统一用 pnpm，它默认敲 npm，装出两份 node_modules；
- 服务器上 pip 必须走内网源，它直连超时后开始乱试参数；
- macOS 上验证过的 brew 命令，到 Linux 容器里全部失效；
- 它为了“修复”环境，直接 sudo 装全局包，污染系统 Python。

代价不只是时间：每轮失败都在消耗上下文，错误的安装动作还可能造成难以回退的环境污染。核心矛盾在于：项目约定可复用，机器事实不可复用，混在一起写，必然有一半失效。

## 做法

1. **确定位置和作用域。** tools.md 放在每台机器的本地工作区，加入 gitignore；仓库里提交一份 tools.md.example 作模板。它只描述“这台机器”，不写项目逻辑。
2. **只写事实，不写愿望。** 按固定分区罗列：系统与架构、运行时版本、包管理器及禁用项、网络与代理、路径约定、已配置的 MCP 服务、明确禁止的动作（如禁止 sudo 全局安装）。
3. **用脚本生成初稿。** 写一个十几行的 doctor 脚本，探测 node/python 版本、已安装的包管理器、代理环境变量，输出 markdown 片段，粘进 tools.md 再补人工约定。机器重装后重跑一遍即可更新。
4. **接进加载链路。** 在 AGENTS.md 里加一句“执行安装/构建类命令前先读 tools.md”，确保 Agent 真的会读它，而不是让它躺在目录里。

骨架示例，每行一条事实：

```markdown
# tools.md — 工作站 A（更新于 2025-06）
- OS: macOS 15, arm64；容器内为 debian bookworm
- Node: 22 LTS，包管理用 pnpm 9，禁止 npm install
- Python: 3.12 via uv，禁止 pip 全局安装
- 网络: pnpm/pip 走 http://127.0.0.1:7890
- 路径: 项目根 ~/work/，venv 一律放项目内 .venv
- 禁止: sudo 装全局包、修改系统 PATH
```

## 踩坑点

- **别放密钥。** tools.md 很容易被截图分享，token 只写“见 .env”。
- **别照抄别人的模板。** 别人写 brew 对你是噪音，按 doctor 的实际输出裁剪。
- **别让它腐烂。** 标注更新日期，定期重跑 doctor；过期信息比没有更糟——Agent 会信任它。
- **别写太长。** 控制在一百行内，Agent 每次会话都可能读它，长文等于持续付费。
- **Windows/WSL 要写清路径转换**（/mnt/c 与 C:\ 的对应关系），否则 Agent 拼出的路径大概率出错。

## 可复用建议

把这套东西固化成三件套：tools.md（机器事实，本地私有）+ doctor.sh（探测与生成）+ tools.md.example（团队模板）。换新机器三步走：跑 doctor、补人工约定、确认加载链路。团队协作时只同步 example 和 doctor 脚本，机器细节各自维护，互不污染。

## 总结

tools.md 解决的不是“Agent 不够聪明”，而是“Agent 不知道你这台机器的事实”。把环境差异从反复的对话试错，挪进一份短的、可验证的、跟机器走的事实清单，试错就变成了查表。保持事实化、简短、带日期、有生成脚本——这套做法不绑定任何框架，在哪台机器上都成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/73b6ea8c27ae8e75.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/8d32b7a345c02a83.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/b50b79883a50a1ca.png)

