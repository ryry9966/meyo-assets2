---
title: LLM 输出解析的防御性编程：JSON 围栏混合格式的分层解析
feedId: 40108
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

凡是让 LLM 返回结构化数据的插件或 Agent 流程，几乎都会撞上同一个问题：提示词里写了“只输出 JSON，禁止 markdown"，模型偶尔遵守、偶尔不遵守。调低温度能缓解，但不能根治——换模型、升版本、改 few-shot 组合，输出习惯就会变。

我们维护一个定时自动化插件时，把线上原始输出留档排查了一段时间，实际形态远比想象的多：裸 JSON、` ```json ` 围栏、无标注围栏、前后带客套话、字符串字段里嵌套代码块。解析处原本只有一句 `json.loads`，某次上游模型升级后成功率明显下滑，才意识到该把“解析”当成独立的工程问题处理。

## 问题

核心矛盾：**协议上你只接受一种格式，现实里你会收到五种以上**。常见变体：

1. 裸 JSON（理想情况）；
2. ` ```json ... ``` ` 围栏包裹；
3. ` ``` ... ``` ` 无语言标注围栏；
4. JSON 前后带说明文字：“好的，以下是结果：{...}"；
5. 某个字段值里含 ` ```python ... ``` `，"取第一个围栏"的策略会抓错对象；
6. 尾逗号、全角引号、BOM 等小毛病。

## 做法

思路是**分层降级**：每层失败进下一层，全程留日志，全部失败时抛带原文的结构化错误。

```python
import json, re

def _fenced(text):
    # 优先带 json 标注的围栏，再退到无标注围栏
    for pat in (r"```json\s*(.*?)```", r"```\s*(.*?)```"):
        for b in re.findall(pat, text, re.S):
            yield b.strip()

def _scan_balanced(text, open_ch, close_ch):
    # 感知字符串与转义的括号配对扫描
    start = text.find(open_ch)
    if start < 0: return None
    depth = in_str = esc = 0
    for i in range(start, len(text)):
        c = text[i]
        if in_str:
            if esc: esc = 0
            elif c == "\\": esc = 1
            elif c == '"': in_str = 0
        elif c == '"': in_str = 1
        elif c == open_ch: depth += 1
        elif c == close_ch:
            depth -= 1
            if depth == 0: return text[start:i+1]
    return None

def parse_llm_json(text: str):
    text = text.strip().lstrip("\ufeff")
    try:                                    # L1 直接解析
        return json.loads(text)
    except json.JSONDecodeError: pass
    for b in _fenced(text):                 # L2 围栏提取，逐块尝试
        try: return json.loads(b)
        except json.JSONDecodeError: continue
    for o, c in (("{", "}"), ("[", "]")):   # L3 平衡扫描
        seg = _scan_balanced(text, o, c)
        if seg:
            try: return json.loads(seg)
            except json.JSONDecodeError: pass
    cleaned = re.sub(r",\s*([}\]])", r"\1", text)  # L4 尾逗号修复
    return json.loads(cleaned)              # 仍失败则抛异常
```

L4 也可以换 `json_repair` 这类修复库，但要清楚它到底“修了什么”。

## 踩坑点

- **贪心正则 `\{.*\}` 会跨对象匹配**。输出里有两段 JSON、或说明文字本身含 `{}` 时直接抓错，非贪婪也解决不了边界问题。
- **朴素括号计数会被字符串内容骗到**。`{"code": "if (a) {b}"}` 里的花括号会让它提前截断，所以扫描器必须跟踪字符串状态和转义。
- **“取第一个围栏”在字段嵌套代码块时会抓到 python 而不是 json**。缓解办法：优先取带 json 标注的围栏，候选块解析失败要继续遍历而不是直接返回。
- **全角引号**在中文场景偶发，可以放在最后一级做定向替换，但别提前——字符串内容本身可能合法包含中文引号。
- **不要静默修复**。每层降级都打点记录，否则模型行为漂移你完全看不见。
- 流式输出别边收边 parse，攒完再走这套流程。

## 可复用建议

- 解析收敛到一个模块，全项目禁止散落 `json.loads`；单元测试用例全部来自线上真实失败样本，攒一个加一个。
- 提示词硬化照做（“只输出 JSON、禁止 markdown、以 { 开头"），但把它当概率手段，解析器才是兜底。
- 能用结构化输出 / JSON mode 就用，但保留同一套 fallback——切换开源模型或供应商降级时你会感谢它。
- 失败时抛带原文的错误对象而不是返回 None，上游才能决定重试还是告警。

## 总结

对 LLM 输出的正确假设是“大概率对、偶尔离谱”。分层降级、全程留痕、真实样本做回归，这套代码半天能写完，但它决定了你的 Agent 是“偶尔抽风需要人盯”，还是“长期稳态可托管”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/2261f5e224e5991b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/114ea1ef431305c2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/8112681f66cd68ba.png)

