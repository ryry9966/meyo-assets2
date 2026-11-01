---
title: LLM 输出解析的防御性编程：一套处理混合 JSON 格式的分层方案
feedId: 43143
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

在 OpenClaw 的插件和自动化流程里，LLM 输出经常被当作结构化数据用：MCP 工具参数、子任务结果回填、多步工作流的状态传递。提示词里写一句"只输出 JSON"，看起来约定清楚了，但实际跑起来格式并不稳定——这一批带代码块，下一批前面多了句解释，再下一批混进全角引号。解析代码只按理想格式写的话，第一天能用，第三天开始在日志里看到成片的 `JSONDecodeError`。

## 问题

常见的"脏输出"大致几类：

1. **代码块包裹**：```json ... ```，有时连语言标注都没有；
2. **前后缀文本**："以下是结果：" / "希望对你有帮助"；
3. **语法瑕疵**：单引号、尾逗号、未转义换行、中文全角引号；
4. **结构混乱**：同屏输出两个 JSON 对象、嵌套 JSON 被二次转义或干脆没转义；
5. **隐性坑**：Python 的 `json.loads` 默认放行 `NaN`/`Infinity`，到下游严格序列化时才炸。

它们是随机组合出现的，且不同模型、不同 temperature 下频率不同。单点修复不够，需要分层。

## 做法：提取 → 清洗 → 解析 → 校验

**第一步：提取候选块。** 不要用 `r'\{.*\}'` 这类贪婪正则——字符串里含花括号、输出里有多个对象时必误伤。用括号配对扫描器：定位第一个 `{`，逐字符计数，取出平衡块。严格实现还要跳过字符串字面量内的花括号，下面从简。

**第二步：按需清洗，不要无条件清洗。** 解析失败才动手：去尾逗号、全角引号换半角。全局暴力替换会破坏字段内容里合法的中文引号，这是实际踩过的坑——用户文本被静默篡改。

**第三步：解析器按成本递进。** 原样 `json.loads` → 清洗后再试 → 宽容解析（json5 之类）→ 最后才是按已知 schema 的字段级正则兜底。每走一层记一笔。

**第四步：schema 校验 + 有限重试。** 用 pydantic/jsonschema 定模型，校验失败把报错拼回提示词重试，上限 1~2 次，再不行走降级默认值，绝不无限循环。

代码骨架十几行：

```python
import json, re

def extract_json_block(text: str) -> str:
    text = re.sub(r"^```[a-zA-Z]*\s*|\s*```$", "", text.strip())
    start = text.find("{")
    if start == -1:
        raise ValueError("no JSON object found")
    depth = 0
    for i in range(start, len(text)):
        if text[i] == "{":
            depth += 1
        elif text[i] == "}":
            depth -= 1
            if depth == 0:
                return text[start:i + 1]
    raise ValueError("unbalanced braces")

def _reject(c):
    raise ValueError(f"non-standard constant: {c}")

def parse_llm_json(text: str):
    candidates = [extract_json_block(text)]
    try:
        candidates.append(clean_common_flaws(candidates[0]))  # 尾逗号/全角引号
    except Exception:
        pass
    for raw in candidates:
        try:
            return json.loads(raw, parse_constant=_reject)
        except (json.JSONDecodeError, ValueError):
            continue
    raise ValueError("LLM output unparseable")
```

数组输出同理，把 `{` 换成 `[` 或两者都扫即可。

## 踩坑点

- **贪婪正则是第一大坑**，嵌套结构和多对象输出必炸，值得单独强调；
- **过度清洗**：弯引号全局替换后，正文里的合法引号也被改掉，数据静默失真；
- **`NaN`/`Infinity`**：`json.loads` 默认接受，用 `parse_constant` 显式拒绝，把问题拦在最上游；
- **修复层掩盖问题**：清洗永远成功，你就永远不知道是提示词的锅。给每条解析路径打计数，命中率高说明提示词该改；
- **temperature=0 不是保险**：只降低频率，长输出、深嵌套下照样翻车。

## 可复用建议

- 封装成统一中间件，所有插件走同一个解析入口，别各写各的；
- 有 function calling / JSON mode / 结构化输出能力时优先用，防御性解析是第二道防线，不是替代品；
- schema 先行：pydantic 模型既做校验，也反向生成 few-shot 示例贴进提示词；
- 定期回看解析路径统计，哪类修复高频，就回去改对应的提示词或换模型。

## 总结

LLM 输出解析没有银弹，正确姿势是把"格式不可靠"当常态来设计：提取、清洗、解析、校验四层独立，每层留观测，重试有上限，降级有兜底。修复层越忙，越说明上游提示词有问题——把它当信号，而不是让它安静地把问题吞掉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/7a5f4dd414845cc4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/1167fcb38caeadf2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/809d38f8b30b9af4.png)

