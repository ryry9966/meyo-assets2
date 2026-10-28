---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42818
source: 综合讨论
publishedAt: 2026-10-28
---

跑 Agent 的人通常不止一台机器：开发笔记本、家里的服务器、一台 VPS，再加上 CI。同一个项目，Agent 在 A 机上用得好好的，切到 B 机就开始瞎猜——Python 在哪个 venv、端口被谁占了、某个 MCP server 到底起没起。我们的做法是把这类"机器事实"收敛到一个 `tools.md` 里，让 Agent 每次会话启动时先读它。它不是又一个 README，而是 Agent 和这台机器之间的契约。

## 问题

把所有信息写进一个文件，会撞上三类麻烦：

1. **环境差异**。绝对路径、版本号、GPU 有无，这些因机器而异，直接提交进仓库就是灾难。
2. **时效性**。手写的配置必然腐化。升级了 Node 忘了改文件，Agent 拿着过期信息继续干活，错误还不好归因。
3. **上下文成本**。把所有细节塞给 Agent，token 花在没用的地方，还稀释了真正重要的指令。

## 做法

核心思路一句话：**不变的进模板，善变的本机生成，每条事实可验证**。分三层：

**第一层：`tools.template.md`（入库）**。只写与机器无关的约定：有哪些工具、命名习惯、调用规范。比如"本地索引服务用 `indexd`，不要自己占 8000 端口"。

**第二层：`tools.local.md`（gitignore）**。只写本机事实：venv 路径、实际版本、监听端口、哪个 MCP server 在跑。不放密钥本体，只放指针（"token 在 `~/.config/xxx`，经环境变量注入"）。

**第三层：探针脚本生成**。与其手写，不如让脚本探测后校验：

```bash
# probe.sh：探测并 diff，差异只提示不覆盖
python3 -c "import sys; print(sys.version.split()[0])"
command -v indexd || echo "MISSING: indexd"
```

挂在会话启动前跑一次，Agent 读到的永远是刚探测过的现实。

**加载顺序**：template → local → 会话内临时覆盖，local 里的同名字段覆盖模板。

**每条事实带验证命令**。这是最值得的一条：与其写 "node 20"，不如写 "node，验证：`node -v` 期望 v20.x"。Agent 拿不准时自己跑一下，而不是猜。

## 踩坑点

- **别让 Agent 随便写 `tools.local.md`**。它可以提议更新，但要人审 diff，否则文件会漂移成它的想象。
- **别写太长**。这个文件每次会话都进上下文，一条事实一行，超过百行就该砍。放不下的细节让 Agent 按需去读别的文档。
- **会话中途改了环境要明说**。Agent 只在启动时读一次，中途升级了依赖就直接告诉它，别指望它重读。
- **CI 里别提交生成物**。CI 现场跑探针脚本生成，仓库里只有模板。
- **路径分隔符**。团队里有 Windows 机器时，模板统一写正斜杠，local 层由脚本归一化，省得 Agent 自己拼错。

## 可复用建议

- 文件分四个 section 就够：`runtime` / `paths` / `services` / `verify`，别发明复杂 schema。
- 把探针脚本当测试用例写：每条事实对应一条探测命令，脚本非零退出码就是环境不达标。
- 换新机器的第一件事是跑探针、生成 local 文件，五分钟搞定，比事后排障便宜得多。

## 总结

`tools.md` 的价值不在文件本身，而在于把"环境事实"从 Agent 的猜测量变成可探测、可验证、分层的输入。模板管不变量，本机文件管差异，脚本管时效，验证命令管兜底。做到这四点，同一套 Agent 配置在几台机器之间切换，就不用再碰运气了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/6a8b3ddf54722b66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/b0c7dc19dde97acf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/1f00111a7289ab68.png)

