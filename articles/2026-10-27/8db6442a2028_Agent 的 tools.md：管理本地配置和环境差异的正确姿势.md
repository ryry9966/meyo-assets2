---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42771
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

跑本地 Agent 一段时间后，最常见的失败不是模型不够聪明，而是工具调用对不上环境：同一份 skill，在公司 Mac 上好好的，回到家里的 Linux 服务器就报 `command not found`；ffmpeg 在 A 机器在 PATH 里，在 B 机器得写全路径；Python 有的是 `python`，有的是 `python3`，还有的必须先 `conda activate`。

这些信息零散地存在于聊天记录、脚本注释和各自脑子里，每次换环境都要重新踩一遍。

## 问题

观察下来，大家处理环境差异基本三种路子，都有毛病：

1. **硬编码在 prompt/skill 里。** 把 `/opt/homebrew/bin/ffmpeg` 写死，换台机器就废，skill 也失去通用性。
2. **让 Agent 现场探测。** 每次都 `which`、`--version` 探一遍。慢、费 token，结果还只活在当次会话里，下次再来。
3. **靠记忆和口头同步。** 团队里 A 的环境知识传不到 B，skill 在别人机器上必然翻车。

本质问题：**机器相关的事实没有落成一份结构化、可维护、Agent 能读的文档。**

## 做法

我们的约定是维护一份 `tools.md`，作为"这台机器长什么样"的单一事实来源。

**1. 定模板，每个条目四行**

```markdown
### ffmpeg
- 用途：音视频转码、抽帧
- 验证：ffmpeg -version
- 调用：/usr/local/bin/ffmpeg（PATH 顺序有问题，不要直接写 ffmpeg）
- 备注：裁 4K 视频记得加 -threads
```

关键是"验证"这一行：给 Agent 一条幂等命令去确认，而不是靠猜。

**2. 分层存放**

- 全局层：`~/.config/openclaw/tools.md`，只放机器级事实（包管理器、Python 路径、代理、GPU）。
- 项目层：仓库根目录 `tools.md`，放项目 venv、私有脚本、数据库连接方式。
- Agent 会话启动时先读全局再读项目，项目条目覆盖同名全局条目。

**3. 写进 Agent 约定**

在系统提示里加两条规则：调用本地工具前先查 `tools.md`；遇到 `not found` 或行为异常时，先核对文档，确认是文档过期就提出更新。这样文档会随使用自愈。

**4. 版本管理**

项目层进 git，全局层进 dotfiles 仓库同步多台机器。

## 踩坑点

- **密钥绝不进 tools.md。** 只写"从 `.env` 读 `XXX_API_KEY`"，文件本身随时可能被贴给模型或进仓库。
- **别写成百科全书。** 只记 Agent 调用需要的信息，人看的用法说明放 README，否则上下文被无效信息吃掉。
- **警惕双份漂移。** tools.md 说一个路径，脚本里又硬编码一个，最后谁也不知道哪个是真的。规则：脚本从环境变量或文档声明的位置读。
- **条目会过期。** 升级工具后路径变了，Agent 还按旧文档走。给条目加"验证命令 + 最近验证日期"，失败两次的条目优先复核。
- **跨平台要分段。** macOS 和 Linux 条目混在一起，Agent 容易拿错。按系统分节，或干脆每台机器一份全局文件。

## 可复用建议

- 梳理方法很简单：翻最近十几次会话，把实际调用过的本地工具列出来就够了，不要贪全。
- 三条铁律：机器事实进全局层，项目事实进仓库，密钥哪都不进。
- 工具调用连续失败两次，就当是文档在报警，先修文档再修环境。

## 总结

tools.md 不是什么精巧设计，就是把"我这台机器和别人的不一样"这件事，从口口相传变成一份 Agent 和人都能读、都会维护的清单。成本十分钟，换来的是 skill 换机器不用改、环境知识不随会话蒸发。在多机、多人协作的场景里，这一步几乎必做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/7e790609ebe1d575.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/8a162a231dac2da2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/f4d21cfcce99459c.png)

