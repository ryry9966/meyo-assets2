---
title: LLM 输出解析的防御性编程：JSON 与标签混合格式的分层容错方案
feedId: 39958
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

写 OpenClaw 插件、接 MCP 工具链或做自动化脚本时，经常需要模型返回结构化数据。虽然多数运行时提供了 tool call 或结构化输出，但实际落地中总有场景要直接解析自由文本：多模型兼容、prompt 里带示例、网关路由到只回纯文本的接口。只要解析过一次你就会发现，"让模型输出 JSON"和"模型输出的是可解析 JSON"是两回事。

## 问题：你拿到的从来不是"纯 JSON"

实际观察到的输出形态，常见有这么几类：

- 裸 JSON（理想情况，反而少见）
- 包在 ```json 代码围栏里
- 包在 `<json>` / `<output>` 之类的 XML 风格标签里
- 前后带一段解释性中文，中间夹着 JSON
- 尾逗号、JS 风格注释、全角引号全角括号混在结构里
- 因 max_tokens 截断的半个对象

任何只按一种格式写的解析器，上线后都会在某天半夜挂掉。

## 做法：四层流水线

思路是分层收口，而不是一个正则打天下：

1. **候选提取**：依次尝试整段文本、代码围栏、标签包裹块、首个配平的 `{...}`；
2. **直接解析**：每个候选拿去 `json.loads`；
3. **轻量修复**：去尾逗号、全角转半角、去注释后重试，重活交给 `json_repair` 这类库；
4. **校验与兜底**：解析结果过 schema 校验；全部失败则把原始输出写日志，并把错误信息回传给模型重试（设上限）。

参考实现（无第三方依赖）：

```python
import json, re

FENCE = re.compile(r"```(?:json)?\s*([\s\S]*?)```", re.I)
TAG   = re.compile(r"<(?:json|output|result)>([\s\S]*?)</(?:json|output|result)>", re.I)

def candidates(text: str):
    yield text.strip()
    yield from (m.strip() for m in FENCE.findall(text))
    yield from (m.strip() for m in TAG.findall(text))
    for opener, closer in (("{", "}"), ("[", "]")):   # 首个配平块兜底
        start = text.find(opener)
        if start < 0:
            continue
        depth, in_str, esc = 0, False, False
        for i in range(start, len(text)):
            c = text[i]
            if in_str:
                if esc:   esc = False
                elif c == "\\": esc = True
                elif c == '"':  in_str = False
            elif c == '"':
                in_str = True
            elif c == opener:
                depth += 1
            elif c == closer:
                depth -= 1
                if depth == 0:
                    yield text[start:i + 1]
                    break

def extract_json(text: str):
    for cand in candidates(text.lstrip("\ufeff")):
        for s in (cand, _repair(cand)):
            try:
                return json.loads(s)
            except json.JSONDecodeError:
                continue
    raise ValueError("no parseable JSON in output")

def _repair(s: str) -> str:
    s = re.sub(r",\s*(?=[}\]])", "", s)                   # 尾逗号
    s = s.translate(str.maketrans("｛｝＂：", "{}\":"))     # 全角标点
    return re.sub(r"^\s*//.*$", "", s, flags=re.M)        # 行注释
```

## 踩坑点

- 贪婪正则 `\{.*\}` 会在字符串内含花括号时抓错区间，用配平扫描或现成库；
- 盲目把单引号替换成双引号，会破坏含撇号的字符串内容，动手前先想清楚；
- 多个代码块时不要默认取第一个——模型可能先复述了你 prompt 里的示例；
- `json_repair` 这类修复库可能"悄悄修好"成结构不对的数据，repair 之后必须再过 schema；
- 重试循环必须设上限并记录原始输出，否则既烧 token 又没法复盘；
- 别指望 temperature=0 或 prompt 里写一句"只输出 JSON"就能一劳永逸。

## 可复用建议

- 把 `extract_json` 沉淀成插件公共工具函数，全项目只维护一份，别每个工具各抄一段；
- prompt 与解析配套设计：要求模型用固定标签包裹输出，同时解析器兼容多种格式，双保险；
- 收集线上解析失败的原始样本，做成回归测试集，每次改格式逻辑都跑一遍；
- 运行时若支持结构化输出或工具调用，优先用，文本解析只做兜底层。

## 总结

防御性解析的核心不是把代码写复杂，而是把不确定性分层消化：提取、解析、修复、校验，每层只处理自己该处理的事。再守住"失败必有日志、日志能定位到具体格式"这条底线，解析层的故障率就能降到可控水平。这套流水线与具体模型无关，换模型、换路由都不用动解析代码。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/f3c4efdb81bab2fd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/64ba9b2ed6c9c23b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/a5f1e1a5fdba89c7.png)

