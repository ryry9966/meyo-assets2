---
title: 让 Agent 帮你写 E2E 测试：一套跑通了的协作流程
feedId: 39666
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

E2E 测试是多数项目里欠账最重的一块：单测覆盖率上去了，关键路径还在靠人肉回归。原因很实际——写一条稳定的 E2E 很费时，选择器易碎、数据准备麻烦、环境一变就红。Agent 加上 MCP 工具之后，这件事有了新解法：让带浏览器能力的 Agent 自己探索页面、起草测试、自己跑、自己修。最近在两个项目里把这套流程跑通了，记录一下。

## 问题

直接让 LLM「生成 Playwright 测试」效果很差，根源是它没有真实页面：

- 选择器靠猜，写出来是 `div:nth-child(3)` 这种一碰就碎的东西；
- 断言过弱，「元素可见」就算过，测试永远绿；
- 不会处理登录态和测试数据，生成的用例根本跑不起来。

一句话：没有工具的 Agent，写的是「看着像对」的测试。

## 做法

流程分五步，跑在 OpenClaw 里，浏览器能力走 Playwright MCP：

1. **给工具，不给猜测机会。** 挂上 Playwright MCP，Agent 能真的打开本地环境、点击、读 DOM、看 console 报错。这是整个方案的前提。
2. **喂约定，而不是只喂需求。** 仓库里放一份测试约定（选择器优先 `data-testid` / role、禁止硬编码 sleep、数据必须走 API fixture），再配 3~5 条人工写的「金标准」用例做 few-shot。
3. **探索 → 起草 → 自跑 → 修复。** Agent 先把流程在页面上走一遍，再写 spec，然后用 `npx playwright test <单文件>` 反复跑到绿。每次迭代只跑一个文件，控制 token。
4. **人工审意图。** diff 只看两件事：断言是否表达了产品预期（而不是照抄当前行为），选择器是否符合约定。Agent 最危险的操作，是把测试改成「能通过一个 bug」。
5. **进 CI，留 trace。** 失败时产出 trace，下一轮让 Agent 拿着 trace 自诊断，比让它重跑猜原因省得多。

## 踩坑点

- **永远绿的测试**：初版生成的断言几乎全是 `toBeVisible`。后来约定强制「每条用例至少一个值断言或反向断言」，明显改善。
- **隐式等待满天飞**：Agent 很爱写 `waitForTimeout(3000)`。约定里明令禁止，只允许 auto-wait 断言，并用 lint 在 CI 拦截。
- **本地绿 CI 红**：登录态没处理。改成 setup 阶段通过 API 登录一次、`storageState` 复用会话，别让每条用例都走 UI 登录。
- **token 失控**：起初让它每轮跑全量，半小时烧掉一大笔。改成 `--grep` 单条跑，全量回归只留给 CI。
- **数据造在 UI 里**：Agent 倾向用页面表单造数据，慢且脆。约定要求 fixture 一律走后端 API。

## 可复用建议

- 测试约定写成仓库级提示文件，任何 Agent 会话自动继承，别靠每次口头交代。
- 人工维护的金标准用例是最值的 few-shot，比长 prompt 管用。
- 生成的测试当代码管：走 PR、走 review、走 CI，不做一次性脚本。
- 建 flaky 记录：隔离后带 trace 让 Agent 修，修不好就删，别让可疑用例污染主套件。
- 分工划清楚：Agent 负责 80% 的脚手架、找选择器、跑通链路；「什么应该为真」必须人定。

## 总结

Agent 写 E2E 的分水岭不在模型多聪明，而在它有没有真实页面的访问权。接上浏览器 MCP、配上仓库约定、用「起草-自跑-修复」的闭环把它圈住，产出就能从「看起来对」跨到「跑得通、敢合入」。人从写测试变成审意图，这笔账在关键路径的回归上，是划算的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/3d494dcc27f3f7d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/900c6bfd1514de6b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/540c86d99db90f2e.png)

