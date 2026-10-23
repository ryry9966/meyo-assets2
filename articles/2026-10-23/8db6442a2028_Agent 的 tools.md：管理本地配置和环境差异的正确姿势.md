---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42348
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

跑 Agent 的人大多遇到过这种场景：同一套 workspace，在公司 Mac、家里的 Linux 服务器和一台 Windows 机器之间切换，agent 一开工就开始猜——用 `brew` 还是 `apt`？docker 装没装？路径是 `/opt/work` 还是 `D:\work`？猜错一次，就是一段失败的 shell 加一段被浪费的上下文。

MCP 和插件解决了"工具怎么接"，但"这台机器上到底有什么、该怎么用"仍需要一个显式载体。tools.md 就是干这个的：一份 agent 开工前必读的环境契约。

## 问题：三个典型反模式

1. **配置硬编码**。系统提示词里写死 `/Users/xxx/...`，换台机器全崩。
2. **事实源混乱**。MCP 配置、shell 别名、tools.md 各描述一遍同一个工具，版本和行为对不上。
3. **文件腐化**。tools.md 变成垃圾场，留着三年前的旧命令，agent 每轮照读，费 token 还误导行为。

## 做法：分层 + 可校验

**1. 拆成两层文件。**
- `tools.md`：进版本库，只写与机器无关的内容——工具用途、适用场景、调用范式、已知陷阱（"不要用 X，用 Y"）。
- `tools.local.md`：进 `.gitignore`，写本机差异——绝对路径、端口、venv 位置、GPU 有无、默认 shell。

**2. 结构固定，方便检索：**

```markdown
# tools.md
## Global Tools    <!-- 跨项目通用 -->
## Project Tools   <!-- 本仓库特有 -->
## Host Overrides  <!-- 引用 tools.local.md -->
## Known Traps     <!-- 明确禁止项 -->
```

**3. Host 段用脚本生成，不靠手写。** 会话钩子跑一次探测：

```bash
for c in docker ffmpeg uv node; do
  command -v $c >/dev/null && echo "- $c: available" || echo "- $c: MISSING"
done
```

输出注入 `tools.local.md`。换机器重跑一遍即可，不依赖人脑记忆。

**4. 把 tools.md 当代码 lint。** 写个校验脚本：解析文中代码块里的命令，逐个 `command -v` 验证存在性；提到的 MCP 工具名与 MCP 配置交叉比对。挂到 pre-commit 和 CI，不一致就报错。这一步能拦掉大部分"文件说有、实际没有"。

**5. 控制 token 预算。** 全文建议 150 行以内；重复步骤封装成脚本，tools.md 只写脚本名和一句话说明，别内联二十行命令。

## 踩坑点

- **别放密钥**。agent 上下文可能被日志记录，tools.md 只写"凭据走环境变量/密钥管理器"，绝不写值本身。
- **标注 shell 类型**。每条命令注明 bash/zsh/pwsh，否则 agent 在 Windows 上会用 bash 语法硬跑。
- **注明最后验证时间**。段末加一行 `<!-- verified: 2025-06 -->`，过期条目在 review 时优先清理。
- **单一事实源**：工具行为描述只留在 tools.md，MCP 配置只管连接参数，互相引用而非重复。

## 可复用建议

- 新项目直接拷四段模板，先填 Known Traps——你踩过的坑，是最该先告诉 agent 的东西。
- 会话开始时让 agent 自检："对照 tools.md 验证环境假设，列出不一致项"。比事后排障便宜得多。
- tools.md 的变更走 PR review，和代码同等对待。

## 总结

tools.md 本质是你和 agent 之间关于环境的一份契约。随手写，它会烂掉并误导 agent；当代码管理——分层、可生成、可 lint、走 review——它就成了跨机器协作里最省心的一层。环境差异永远存在，正确姿势不是消除差异，而是把差异收敛到一个显式、可校验的文件里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/57b431d4feba038d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/fa5f8bc70d1531c8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/a242f0cb6d803a4d.png)

