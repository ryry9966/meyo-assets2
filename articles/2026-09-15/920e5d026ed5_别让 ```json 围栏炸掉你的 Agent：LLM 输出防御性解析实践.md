---
title: 别让 ```json 围栏炸掉你的 Agent：LLM 输出防御性解析实践
feedId: 37667
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

在 OpenClaw 里写 skill、插件或 MCP 工具时，让模型返回结构化 JSON 是最常见的需求。提示词里明明写了"仅输出 JSON，不要解释"，线上拿到的输出却五花八门：` ```json ` 围栏、围栏前多一句"好的，以下是结果"、XML 风格标签（如 `<tool_call>{...}</tool_call>`）和 JSON 混排、同一次输出里两个代码块并存。解析一旦失败，整条 agent 链路就断在那里。这篇帖分享一套踩坑后沉淀下来的分层解析做法。

## 问题

失败模式归纳下来主要有五类：

1. markdown 围栏包裹，语言标注可能是 `json`、`JSON`、空或别的；
2. 围栏外夹带开场白/结束语，或被 XML 风格标签包了一层；
3. 输出中有多个代码块（先示例后正式结果）；
4. JSON 本身不规范：尾逗号、单引号、`True/False/None`、`//` 注释、智能引号；
5. JSON 字符串值内部本身包含 ` ``` `，导致正则截断。

对应的脆弱写法是 `json.loads(text.strip())` 或简单 `replace("```", "")`——本地 demo 都能过，换个模型、换个采样就翻车。

## 做法：分层解析管线

核心思路：按成本从低到高逐层尝试，任一层通过即返回，全部失败再走重试。

**第 1 步，生成候选。** 去 BOM 和零宽字符；先剥掉已知的包裹标签（`<tool_call>` 等）；用正则抓出全部围栏块；没有围栏时，取第一个 `{`/`[` 到最后一个 `}`/`]`。原文永远作为第一个候选：

```python
import json, re

FENCE = re.compile(r"```[\w-]*\s*\n(.*?)```", re.DOTALL)

def candidates(text: str) -> list[str]:
    text = text.lstrip("\ufeff").strip()
    blocks = [m.group(1) for m in FENCE.finditer(text)]
    if not blocks:  # 无围栏：取首 {/[ 到末 }/]
        i = min((text.find("{"), text.find("[")),
                key=lambda x: x if x != -1 else 1 << 30)
        j = max(text.rfind("}"), text.rfind("]"))
        if 0 <= i < j:
            blocks = [text[i:j + 1]]
    return [text, *blocks]

def parse_llm_json(text, fixes=(str, strip_commas, normalize_literals)):
    for c in candidates(text):
        for fix in fixes:
            try:
                return json.loads(fix(c))
            except Exception:
                pass
    raise ValueError("unparseable: " + text[:200])
```

**第 2 步，严格优先、修复其次。** 修复链只做保守变换：去尾逗号、智能引号转直引号、字面量归一化。也可以直接用 json5 / json_repair 这类现成库，别自己造轮子造出 bug。

**第 3 步，Schema 校验。** `loads` 成功不等于能用。用 pydantic（或 zod）校验字段存在性与类型，校验失败按解析失败处理。

**第 4 步，带反馈重试。** 把上次的原始输出和报错信息拼回 prompt，明确告诉模型哪里不合法，重试 1–2 次；仍失败则落盘原始输出、走降级分支，不要让异常上抛断掉整条任务。

## 踩坑点

- **围栏正则的陷阱**：`.*?` 非贪婪匹配遇到字符串值里的 ` ``` ` 会提前截断，所以候选要"围栏块 + 括号截取 + 原文"一起试、逐个 `loads`，别只信第一个。
- **多个代码块**：只取第一个块不够，模型有时先给错误示例再给正确答案。用 schema 校验是否通过来挑选，比猜位置可靠。
- **修复要克制**：智能引号替换可能改坏正文字段，永远先试严格解析，修复只对已失败的候选做。
- **保留原始输出日志**：统计解析失败率，能定位是哪个模型、哪类 prompt 引发，比事后凭感觉改提示词有效得多。

## 可复用建议

- 解析器封装成公共模块，skill、插件、MCP 工具统一引用，不要各写各的 replace；
- 提示词约束加低温度采样仍是第一道防线，防御性解析是兜底而非替代；
- 模型端支持 JSON mode / 结构化输出就优先用，本地管线做第二道保险；
- 解析失败统一埋点上报，失败率本身就是模型与提示词质量的观测指标。

## 总结

不要假设模型会遵守格式要求，要假设它"大概率听、偶尔不听"。把格式不确定性当成正常输入来设计：候选生成、严格优先、保守修复、schema 校验、反馈重试、日志回放。这套管线在我们几个长期运行的 OpenClaw 自动化任务里，把解析相关的断链基本压到了零，可以直接抄走改。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/eaa6b4ad52415040.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/acaf744b8e8fcc36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/bac564b88d48b2fe.png)

