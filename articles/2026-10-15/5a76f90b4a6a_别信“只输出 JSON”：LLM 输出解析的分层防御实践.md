---
title: 别信“只输出 JSON”：LLM 输出解析的分层防御实践
feedId: 41601
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

写 OpenClaw 插件或自定义 skill 时，让模型输出结构化 JSON 是高频场景：子代理回传结果、skill 内两段式调用、MCP 工具的前置参数抽取，都绕不开“解析模型输出”这一步。提示词里写一句“只输出 JSON，不要多余文字”，多数时候有效；但只要跑得够久、切过的模型够多，解析失败一定会来。

## 问题

从实际失败样本看，大概几类：

1. Markdown 围栏包裹：```json ... ```，围栏里偶尔还混着说明文字；
2. 自定义标签包裹：`<json>`、`<result>`、`<output>`，甚至标签里再套一层围栏；
3. JSON 前后带一句“好的，以下是结果：”；
4. 语法瑕疵：尾逗号、全角引号/冒号、`//` 注释、单引号键；
5. 输出多个 JSON（一段复述的示例 + 一个正式结果）。

裸的 `JSON.parse` 只覆盖其中一种。更麻烦的是 OpenClaw 支持切换多个模型，不同模型的格式习惯不同，写死单一解析格式等于把逻辑和某个模型绑死。

## 做法

思路：解析做成一条分层管道，逐层降级，并记录每层命中情况。

```ts
export function extractJson(raw: string): unknown {
  const text = raw.replace(/^\uFEFF/, "").trim();
  try { return JSON.parse(text); } catch { /* L0 失败，降级 */ }

  const candidates: string[] = [];
  // L1 围栏：用 `{3} 写法，避免和代码块本身冲突
  for (const m of text.matchAll(/`{3}(?:json)?\s*([\s\S]*?)`{3}/g))
    candidates.push(m[1]);
  // L2 自定义标签（大小写不敏感）
  for (const m of text.matchAll(/<(json|result|output|answer)>\s*([\s\S]*?)\s*<\/\1>/gi))
    candidates.push(m[2]);
  // L3 括号配平扫描，兜底纯文本包裹、多对象
  candidates.push(scanBalanced(text));

  for (const c of candidates)
    for (const r of repair(c)) // repair 返回风险从低到高的多个修复版本
      try { return JSON.parse(r); } catch { /* 继续 */ }

  throw new Error("no parseable JSON found");
}
```

几个实现要点：

- `scanBalanced` 用栈扫描 `{}` / `[]` 配平，必须维护“当前是否在字符串内 + 转义”的状态机，否则字符串内部的引号和花括号会把配平彻底搞乱；
- `repair` 只做低风险修复：去尾逗号、全角标点转半角、剥离字符串外的 `//` 注释；单引号转双引号风险最高，放到最后一档；
- 多候选时不要无脑取最后一个，按“可解析成功”优先取第一个；经验上“靠后且可解析”的那份往往才是正式结果。

解析成功后别直接用，接一层 schema 校验（ajv/zod 均可）。校验失败时，把具体报错拼回提示词重试一次——带反馈的一次重试，远好于盲重试三次。

## 踩坑点

- 正则贪婪匹配会跨代码块，务必非贪婪 + `matchAll` 收集全部候选；
- 提示词里放了 JSON 示例，模型可能在输出里复述它，于是出现两个 JSON；
- 盲目替换引号会破坏字符串内容本身就含单引号的数据；
- 解析失败只抛异常不留现场，事后没法排查——先把原始输出落盘（脱敏），再抛错；
- 别指望 JSON mode 万无一失，部分模型/网关对 structured output 支持不完整，兜底管道必须常驻。

## 可复用建议

- 把 `extractJson` + schema 校验 + 带反馈重试封装进插件模板，所有 skill 共用；
- 日志里记录“命中第几层”：L3 配平扫描命中率持续偏高，说明该改的是提示词，不是继续堆解析逻辑；
- 把线上真实失败样本脱敏后存成测试 fixture，解析器每次改动跑一遍回归；
- 提示词和解析器是同一个契约的两面，改任何一边都跑同一组用例。

## 总结

把模型输出当不可信输入处理，是插件工程和聊天 demo 的分水岭。分层解析、schema 校验、带反馈重试，三件事都不复杂，但能把“偶发解析失败”变成“可观测、可回归”的常规工程问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/861e7083a51070ea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/6a7f4df889b73492.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/5ee8247f153c6836.png)

