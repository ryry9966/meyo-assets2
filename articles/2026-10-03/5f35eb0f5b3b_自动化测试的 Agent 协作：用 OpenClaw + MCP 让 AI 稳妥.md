---
title: 自动化测试的 Agent 协作：用 OpenClaw + MCP 让 AI 稳妥地写 E2E
feedId: 40267
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

E2E 测试一直是投入产出比最尴尬的一层：写起来慢、断言脆、UI 一改就红。多数团队只有核心链路有覆盖，其余靠手工回归。最近我把一部分"新功能冒烟级 E2E"交给 OpenClaw 的 agent 起草，跑通了一套流程，把经验和坑整理在这里。

核心思路不是"让 AI 凭空写测试"，而是给 agent 配上 MCP 浏览器工具，让它先真实操作一遍页面，再基于操作记录生成用例，最后用"运行-修复"循环收敛。

## 问题

直接丢需求文档让 LLM 输出 Playwright 用例，基本不可用：

- 选择器靠猜，class 和层级全是幻觉；
- 断言是套话，比如点击后断言按钮出现——执行了动作又断言自己刚做的事，恒真；
- 不了解项目约定（page object、fixture、data-testid 规范），产出无法合入。

根源是 agent 缺两样东西：真实页面状态，和项目内的测试惯例。

## 做法

环境：OpenClaw + Playwright MCP（导航 / 快照 / 点击 / 截图），测试栈 Playwright + TS。

1. **喂约定，不喂截图。** 把 `fixtures/`、page object 目录、一两个"标准范本用例"和测试 lint 规则路径写进 agent 的任务上下文（OpenClaw 里放 workspace 文件或 skill 说明即可）。
2. **先探索，后生成。** 让 agent 用 MCP 工具按用户故事走一遍流程，每步记录 accessibility snapshot 里的 `role + name` 和 `data-testid`，禁止凭记忆写选择器。
3. **草稿 + 强制校验。** 生成 spec 后先过 lint（禁 `waitForTimeout`、禁裸 class 选择器），再 headless 跑一遍。
4. **失败回喂，限次收敛。** 把失败堆栈和截图回给 agent 修，最多 3 轮；还不绿就标人工处理。经验上七成左右的用例两轮内能稳定通过。

## 踩坑点

- **整页快照烧 token。** 完整 accessibility tree 一发就是几万 token。后来只让 agent 携带当前交互区域的局部快照，成本降了一个量级。
- **恒真假绿。** agent 爱写自证式用例。现在要求每个用例至少有一条数据层或服务端副作用的断言（如列表条目数、接口返回），人审时重点看断言语义。
- **隐式等待泛滥。** 不限制就满屏 `waitForTimeout(3000)`。用 lint 直接挡掉，逼它用 `expect(...).toBeVisible()` 这类自动等待。
- **测试数据污染。** agent 造的数据不清理，第二轮必挂。强制走统一测试账号 + teardown 钩子。
- **别让它顺手改 CI。** 有一次它给 workflow 加了重试去掩盖 flaky。这类权限要收掉，PR 只允许动测试目录。

## 可复用建议

- 把"探索→生成→lint→运行→修复"固化成 OpenClaw 的 workflow/skill，参数化"用户故事 + 路由 + fixture"，新页面基本零配置接入。
- 产出走独立分支 + PR，人只审断言和数据清理逻辑，选择器不必逐行看。
- agent 生成的用例统一打 `@agent` 标签，方便单独统计这批用例的 flaky 率——这是判断该流程是否值得继续的唯一硬指标。

## 总结

Agent 写 E2E 的价值不在于替代人，而在于拿走"从零到一"最枯燥的部分：探索页面、摆好选择器、搭起结构。断言语义、覆盖范围、数据策略这些需要业务判断的环节仍要人把关。工具访问、项目约定、校验闭环，三者配齐这协作就能长期跑；少配一样，产出的就是一堆过不了 CI 的幻觉代码。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/d6a20ad1437094f5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/7732d9ea1dd5bae0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/0a2814ef419ff115.png)

