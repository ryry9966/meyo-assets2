---
title: LLM 输出解析的防御性编程：JSON 围栏与脏输出的分层清洗
feedId: 39372
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

OpenClaw 的插件和 skill 开发里，让模型吐结构化 JSON 是高频需求：任务路由、参数抽取、批处理判定。走 structured output 或 JSON mode 固然稳，但实际经常绕不开裸解析——本地模型不支持 JSON mode、第三方兼容端点、CLI 子命令拿到的就是一段纯文本。这时 `JSON.parse` 的成功率，就是你自动化流水线的成功率。

## 问题

同一段 prompt，实测输出有这么几类变体：

1. 直接可解析的裸 JSON；
2. 包在代码围栏里、带 `json` 语言标识；
3. 围栏没有语言标识；
4. 前后带"好的，以下是解析结果："之类的话；
5. 推理模型先吐 `<think>` 段落再给 JSON；
6. 尾逗号、弯引号、字符串里未转义的换行。

任何一种都会让裸 parse 抛异常。无人值守场景下表现为任务静默失败，复现率 3%–8%——低到不会被当成 bug 优先修，高到每天都能撞上。

## 做法

核心思路：不做一次性的"聪明"正则，做逐级降级的清洗管线。

````js
function robustParse(raw) {
  let text = String(raw).replace(/<think>[\s\S]*?<\/think>/g, '');
  // 非贪婪，只剥最外层围栏
  const fence = text.match(/```(?:json)?\s*([\s\S]*?)```/);
  if (fence) text = fence[1];
  // 截取首个 { 或 [ 到末个 } 或 ]
  const s = text.search(/[{[]/);
  const e = Math.max(text.lastIndexOf('}'), text.lastIndexOf(']'));
  if (s >= 0 && e > s) text = text.slice(s, e + 1);
  try {
    return JSON.parse(text);
  } catch {
    return JSON.parse(
      text
        .replace(/[\u201c\u201d]/g, '"') // 弯引号
        .replace(/,\s*(?=[}\]])/g, '')   // 尾逗号
    );
  }
}
````

parse 通过不代表结束：用 zod / jsonschema 校验字段与类型，"解析成功但结构不对"的输出实测并不少。最后留一条自愈路径——把原始输出和报错喂回模型重试一次，仍失败则落盘告警，由人工或上游策略接管。

## 踩坑点

- **围栏正则必须非贪婪。** JSON 字符串值里自带 markdown 代码块的情况很常见，贪婪匹配会把两个独立代码块粘成一坨。上面只取第一个完整围栏是刻意的。
- **引号替换要克制。** 全局把单引号换双引号是经典翻车点——字符串里合法的撇号会被打碎。只修确定性错误，其余交给自愈重试。
- **修复放在边界截取之后。** 先全局替换再定位边界，可能把正文里的引号也误改。
- **流式场景不要边收边 parse。** 攒到结束帧再走管线，否则处理的是"半个 JSON"。
- **失败必须落原始输出全文**，不能只记 "parse failed"。事后排障全靠这个。

## 可复用建议

- 封成一个 util 模块，skill、插件、MCP 工具统一引用，别各写各的。
- 用真实失败样本建 fixture 目录，parser 每次改动跑回归。脏输出是长尾，攒样本就是攒护城河。
- prompt 侧同向发力："只输出 JSON，不要解释文字"，附一个最小示例，能压掉大半废话和围栏。防御性解析是兜底，不是第一道防线。
- parse 成功 ≠ 业务成功：schema 校验、必填字段、枚举值都要过一遍。

## 总结

LLM 输出本质上是非受控输入。与其赌它这次守规矩，不如假设它总以你没见过的方式不守规矩。这条"剥思考标签 → 剥围栏 → 截边界 → parse → 定点修复 → schema 校验 → 自愈重试"的管线不到四十行，换来的是自动化失败率从"偶尔抽风"降到"几乎无感"。这类脏活不值得炫技，但非常值得一次性做扎实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/1f6762ac528cd520.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/41d037f02901c85a.png)

