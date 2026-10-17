---
title: OpenClaw 定时任务选型：cron 还是 heartbeat？
feedId: 41867
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

OpenClaw 里有两个容易被当成"同一种东西"的定时机制：

- **cron**：由 Gateway 调度器触发，按 cron 表达式或一次性 `--at` 时间启动一次 agent 运行，默认跑在隔离会话里，输出可以 announce 投递到 Telegram 等渠道。
- **heartbeat**：主会话按固定间隔（默认 30 分钟）被唤醒一次，读工作区的 `HEARTBEAT.md`，判断有没有该做的事；没事就回 `HEARTBEAT_OK`，这条回复会被吞掉，不打扰你。

一个像排班表，一个像值班心跳。混用的代价是：要么重复通知，要么 token 烧了没产出。

## 问题

我踩过的三类典型误用：

1. 把所有轮询都塞进 `HEARTBEAT.md`，主会话上下文又大，账单难看；
2. 把"结合我们之前聊的方案"这类依赖上下文的判断写进 cron，隔离会话根本不知道"之前"是什么，答非所问；
3. 容器没设时区，任务整整晚 8 小时触发，还以为调度坏了。

## 做法

**第一步：按三个维度分类任务。**

- 是否必须精确到点（提醒、对外交付）→ cron
- 是否依赖主会话上下文（"盯着那件事"）→ heartbeat
- 高频轻量轮询 → cron 隔离会话；低频判断类 → heartbeat

**第二步：确定性任务进 cron。**

```bash
# 周期任务：工作日早上出简报并投递
openclaw cron add \
  --name "daily-brief" \
  --cron "30 9 * * 1-5" \
  --message "汇总昨天的 commits 和未关闭 issue，输出不超过 5 条要点" \
  --announce --channel telegram --to "<chat-id>"

# 一次性提醒
openclaw cron add --name "pack-luggage" --at "2026-03-14T21:30" --message "提醒我收拾行李"
```

message 必须自包含：上下文、动作、输出格式全写全，别依赖历史。改完先 `openclaw cron run <id>` 手动触发验证，再看 `openclaw cron runs <id>` 的运行历史。

**第三步：语境型巡检进 heartbeat。**

在 `HEARTBEAT.md` 里写"条件 + 动作"清单，不要写散文：

```markdown
- 若 ~/deploy/flag.done 存在：读取内容，用 TG 汇报结果，然后删除该文件
- 检查 GitHub Actions 最近一次运行，失败则提醒我
- 以上都不满足：回复 HEARTBEAT_OK
```

唤醒频率在配置里调（字段以 `openclaw doctor` 和配置文档为准），比如 45 分钟一次，并限定活跃时段。

## 踩坑点

- **时区**：容器默认 UTC，给 Gateway 设 `TZ=Asia/Shanghai`，否则 cron 全部错位。
- **heartbeat 成本**：每次唤醒都带着主会话全量上下文计费，哪怕最后只回 HEARTBEAT_OK。频率 × 上下文长度就是账单，清单务必短、条件化。
- **隔离会话没有记忆**：cron 的 message 里别写"如前所述"；确实需要主会话的场景才显式切换，并接受成本。
- **错过不补跑**：Gateway 停机或笔记本合盖期间，cron 和 heartbeat 都会静默错过。重要任务放常驻设备，外部再用 Uptime 监控兜底。
- **双轨重复**：同一件事别既在 cron 又在 HEARTBEAT.md，会收到两份通知。

## 可复用建议

- 一句话选型：**到点必须做、做给别人看 → cron；看着办、带上下文、轻量判断 → heartbeat。**
- `HEARTBEAT.md` 控制在 10 行以内，每行都是"如果 X 则做 Y"。
- cron 的 message 当成给新同事的完整工单写。
- 每周看一次 `cron runs` 和会话成本：高频 heartbeat 项降级成 cron，没用的 cron 直接删。

## 总结

cron 是排班表：精确、隔离、可审计，适合交付类任务；heartbeat 是值班心跳：带着记忆巡检，适合"盯着点"类判断。选型错了不是跑不起来，而是成本和可靠性都会变差。把确定性交给 cron，把语境判断交给 heartbeat，再严格控制 heartbeat 的频率和清单长度，定时任务就能长期稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/fb8ba0fbf9609b46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/86d3cdee22ed04fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/295d283a5566bac2.png)

