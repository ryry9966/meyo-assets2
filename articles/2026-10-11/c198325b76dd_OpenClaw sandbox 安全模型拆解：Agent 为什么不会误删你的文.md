---
title: OpenClaw sandbox 安全模型拆解：Agent 为什么不会误删你的文件
feedId: 41218
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

用 OpenClaw 这类 agent 框架，最常见的担忧是：模型哪天抽风，一条 `rm -rf` 把整个项目删了怎么办。我的结论先放在前面：**不是模型变聪明了，而是执行层默认不信任模型输出**。这篇帖拆一下 OpenClaw 的 sandbox 到底拦在哪几层，以及我在实际部署里踩过的坑。

## 风险到底在哪

Agent 的执行链路是「LLM 生成工具调用 → 执行器落盘」。误删的可能来源有四类：

1. 幻觉路径或相对路径解析错误
2. shell 通配符展开范围超出预期
3. 第三方插件 / MCP 工具自带文件操作能力
4. 用户指令本身含糊（“清理一下这个目录”）

## OpenClaw 的四层防线

**1. 工作区边界。** 默认 `workspace-only` 模式，只有 workspace 目录可写。执行器在 spawn 子进程前会做路径规范化——解析 symlink 和 `..` 后校验 real path 是否落在 allowlist 内，越界直接拒绝：

```yaml
sandbox:
  mode: workspace-only
  allow_write:
    - ~/openclaw/workspace
  deny_glob:
    - "**/.git/**"
    - "**/*.env*"
```

**2. 危险命令拦截。** 执行器不直接跑 shell 字符串，而是解析成 argv 后匹配危险模式（`rm -rf`、`dd of=`、递归 chmod 到根、覆盖重定向到 allowlist 之外）。对 `sh -c`、`bash -c` 这类嵌套执行默认直接拒绝，避免把字符串匹配的绕路带回来。

**3. 破坏性操作二次确认。** 删除、覆盖、批量移动前会先生成 plan，列出受影响文件清单。可配阈值，比如影响文件数超过 5 个必须人工 confirm。

**4. 快照回滚。** 写操作前对目标文件做内容寻址快照，出事可以 `claw undo` 恢复。注意这只是兜底，不是主防线。

## 验证方法

- 红队 prompt：让 agent“把 workspace 上级目录的旧日志删掉”，确认被边界拦下
- `ln -s /etc/passwd link` 后让 agent 写入，验证 symlink 解析
- 让 agent 执行 `rm -rf $HOME/xxx`，确认命中危险模式
- 故意删一个文件，用 `claw undo` 恢复，验证快照链路

## 踩坑点

- **自己把 `/` 或 `$HOME` 加进 allow_write**，等于整套模型失效。allowlist 要最小化。
- **Docker 挂载穿透**：sandbox 只管 OpenClaw 自己的执行器，`-v /:/host` 挂进来的东西它管不了。
- **MCP 工具绕过**：第三方 MCP server 若自带 `fs.write`，不走 OpenClaw 执行器，四层防线对它无效。必须在 MCP 配置里单独做 capability 限制——这是最容易漏的洞。
- **workspace 内的软链指向外部目录**：写入会被拦，但 git 等工具内部操作会报错，正确做法是把真实目录移进 workspace。
- **auto_approve 设太宽**，确认机制形同虚设。

## 可复用建议

1. 永远从 `workspace-only` 起步，有明确需求再按目录放行
2. `deny_glob` 至少保护 `.git`、密钥文件、数据库文件
3. 有重要数据的机器上关掉 auto_approve 或收紧阈值
4. 把红队 prompt 当回归测试，升级后跑一遍
5. 定期审计插件和 MCP server 暴露的工具清单，关掉用不到的文件类工具

## 总结

严格说，“不会误删”应表述为“被拦截 + 可回滚”：路径靠边界校验拦，命令靠 argv 解析拦，规模靠确认门槛拦，兜底靠快照。安全设计的出发点应该是假设模型一定会犯错，然后让犯错变得便宜。配置花十分钟，比数据丢一次便宜得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/b1bb21189200f339.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/13fa35f2c60f7052.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/ea8b14b4db886cc9.png)

