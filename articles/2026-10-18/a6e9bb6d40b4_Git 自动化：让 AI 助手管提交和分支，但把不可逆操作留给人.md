---
title: Git 自动化：让 AI 助手管提交和分支，但把不可逆操作留给人
feedId: 41853
source: 综合讨论
publishedAt: 2026-10-18
---

# 背景

日常开发里有两件最磨人的琐事：写符合团队规范的 commit message，以及清理本地越堆越多的分支。前者重复枯燥，后者极易遗忘。OpenClaw 这类常驻 agent 配上 shell/MCP 工具后，很自然想到让它接手。但 git 是一把既能干活也能切手的锯子——`push --force`、`reset --hard`、误删未合并分支，任何一次“AI 自作主张”的代价都不小。这篇文章记录我们摸索出的分级授权方案。

# 问题

- commit message 质量靠自觉，风格漂移严重，review 时来回扯皮；
- 本地 30+ 分支，分不清哪些已合并、哪些还活着；
- 最初直接把 `git *` 开放给 agent，一周内出现过 `git add -A` 把构建产物和 `.env` 一起暂存的事故，从此不敢放养。

# 做法：分级授权 + 脚本护栏

核心思路：能力分级，危险操作永远留给人。

**第一步，只读起步。** 先只开放 `status / diff / log / branch -a`，让 agent 干两件事：读 staged diff，起草 Conventional Commits 风格的 message；每周扫一次分支，输出“已合并可删 / 长期未动”清单。零风险，收益已经很明显。

**第二步，用包装脚本而非裸 shell 给写权限。** 我们写了个 `gitsafe` 做命令网关：

```bash
case "$*" in
  *push*--force*|*reset*--hard*|*branch*--force*)
    echo "blocked: destructive op needs human" >&2; exit 1;;
  *add*-A*) echo "blocked: use explicit file list" >&2; exit 1;;
esac
exec git "$@"
```

agent 只允许调 `gitsafe commit -m ...`，且 message 必须引用 diff 里真实出现的改动。push 一律人工触发。

**第三步，分支清理流程化。** agent 生成清单 → 人工勾选 → 人工执行删除。agent 负责“发现”，不负责“执行”。

# 踩坑点

- **交互式命令会挂死**：`rebase -i`、需要输密码的 https remote，在非交互 shell 里直接卡住。务必设超时，并规避交互入口。
- **secrets 进了上下文**：diff 带着 `.env` 的 token，agent 会原样复述到日志。敏感文件要在工具层排除，而不是在 prompt 里求它“别看”。
- **message 幻觉**：AI 偶尔描述不存在的改动。要求它先引用 hunk 再总结，能压掉大部分编造。
- **多仓库路径串了**：agent 同时挂几个项目时，在错误的 repo 里 commit 过。现在每个任务显式传仓库绝对路径。
- **别让它碰 origin**：本地怎么折腾都能恢复，推上去的误操作要靠 revert 补救，成本完全不同。

# 可复用建议

1. 权限从只读到可写逐级放开，每级观察一周再升级；
2. 护栏写在脚本/工具层，别指望 prompt 约束——prompt 会被长上下文稀释；
3. agent 的所有 git 操作落日志，出问题能回放；
4. 分支命名、commit 格式固化成一个技能模板，让 agent 输出前自查；
5. agent 的配置本身也进 git 管起来，护栏改动可追溯。

# 总结

AI 助手很适合承接 git 里“看”和“起草”的部分：读 diff、写 message、盘点分支、汇总 changelog。而不可逆的按钮——force push、删分支、改写历史——留在人手里。十几行 case 语句换来的安心感，非常值。自动化的目标不是全自动，而是把人的注意力从琐事挪到真正需要判断的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/844989e13ad811e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/fcf9b97ba1ecc1e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/c96938882ae4afdb.png)

