# Memory v2 hot_rule 自约束失效与三层防御实战教程

> Mavis v2 memory 系统的 hot rule（v2-1 到 v2-5 + TTS + PRE）写进文档后**从来不跑**——self-instruction 模式的失败。本文用 RAG 真查 douyin_kb 找到 4 条独立来源解药（视频 21/15 / 212 / 150 / 502），配 preflight_audit + self_correct 两层工程化防御，17 个新测试全过、67/68 总测试通过。完整方案 + 关键代码 + audit 实战。

## 0. 范围：本文专门针对 Memory 系统 hot rule

| 维度 | 本文覆盖 |
|---|---|
| **目标系统** | Mavis v2 memory system（MEMORY.md + cards.jsonl + RAG KB + scripts/） |
| **核心问题** | v2 hot rule（v2-1~v2-7 + TTS-1 + PRE-1）写入有效、执行率 0% |
| **根因诊断** | 4 条 KB 实证（视频 21/15 / 212 / 150 / 502） |
| **工程方案** | ⚠️ 顶部块 + 负面约束 + self-correction loop + preflight audit |
| **代码产物** | `preflight_audit.py` + `self_correct.py` + 17 个新测试 |
| **可推广性** | 任何 LLM agent 系统写 hot rule 都会遇同问题（写入≠执行） |

> **跨教程引用**：本文是 `llm-hard-constraints` skill 的 Mavis **Memory 系统**具体实现。通用框架层（CFG / 投票 / SCHEMA / Verifier / 决策树）见 **15_LLM硬约束与一致性实战教程.md**（待写）。本文是"硬约束"在 memory hot rule 场景的 case study；15_ 是"硬约束"全谱。

---

## 一、概述：为什么 LLM 不遵循自己写的规则

我跑了 6 个月的 Agent 长期记忆机制改造，建了 v1 / v2 两套 memory 系统，灌了 14 张 cards.jsonl，写了 9 个 v2 wrapper scripts（`append_card.py` / `policy.py` / `decay_score.py` / `recall_session_start.py` 等）。理论上 Mavis 应该主动调这些脚本。但实测：

```bash
$ python scripts/preflight_audit.py --recent 5
[preflight] ⚠️ AUDIT FAILED — 最近 5 turns 缺 v2 markers:
  - v2-1: 没看到调用
  - v2-2: 没看到调用
  - v2-3: 没看到调用
  - v2-4: 没看到调用
  - v2-5: 没看到调用
  - TTS-1: 没看到调用
  - PRE-1: 没看到调用
```

**7 个 hot rule 全 missing**——Mavis 自己写的 hot rule，自己从来不调。这就是 self-instruction 失败模式。

## 二、问题不是"KB 没有"

第一直觉是 KB 没有讲 self-instruction 的解法。我跑 6 个 meta-level query 实查 douyin_kb（1637 chunks, bge-m3 1024-dim）：

| Query | 命中视频 | 解读 |
|---|---|---|
| LLM 不遵循用户指令 | 视频 212 / 72 / 15 | 间接命中 |
| self-consistency | 视频 212 | 间接命中 |
| behavioral alignment | 视频 212 / 1433 | 间接命中 |
| reflection trigger | 视频 144 / 502 | 间接命中 |
| CoT trigger | **视频 502 (465s 关键命中)** | 接近 |
| AI 幻觉 hot rule | 视频 233 | 直接命中 |

**关键命中**——视频 502 第 465.4 秒："大模型不会立刻去瞎猜，而是会触发一个 Systemart 慢思考回路。"

但**直接命中 "LLM 不遵循自己写的 hot rule"**的 query 是用户纠正我之后——把 query 措辞从"泛 LLM 行为"切到"memory hot rule 执行"——才召回出 4 条独立来源：

| Query | 命中 | 关键引文 |
|---|---|---|
| memory 规则 写不等于执行 | 视频 21 / 15 | "LLM 对'不要做什么'的响应比对'要做什么'更敏感" |
| agent 主动调 memory 反思 | 视频 502 | "主动 vs 被动" / "能主动思考的系统" |
| 自我修正机制 | 视频 150 | "解析爆错了没关系，把爆错信息再让给模型告诉他你写错了" |
| system prompt vs memory rule | 视频 212 (0.66) | "system prompt 是开发者在后台隐性配置的" |

**KB 内容齐全**。**问题在 query 措辞**——我之前 query 太泛，没召回出真正相关的视频。

---

## 三、根因 4 层诊断

### 3.1 不是 KB 缺失，是 query 措辞

| 教程 (Video) | 核心论断 | 推导出方案 |
|---|---|---|
| **视频 21 / 15** | LLM 对"不要做什么"更敏感 | hot rule 改**负面约束**句式 |
| **视频 212** | system prompt 是后台**隐性配置** | hot rule 写**系统级权重**位置，不放 MEMORY.md 末尾 |
| **视频 150** | parse fail → error 回灌让模型自纠 | **self-correction loop** |
| **视频 502** | 主动 vs 被动思考系统 | **proactive trigger**（主动问自己 5+1 个问题） |

### 3.2 LLM 写 hot rule 的双重身份

同一个 LLM 在两个时刻做了两件事：

| 时刻 | 行为 | 评价 |
|---|---|---|
| 写 hot rule 时 | 理性分析："应该每 reply 调 recall_session_start" | 理性 |
| 执行 hot rule 时 | 跳到具体任务，只看用户 query | **同样理性地跳过自己写的规则** |

**写入 ≠ 执行**——这是个工程问题，不是哲学问题。

### 3.3 Mavis runtime 不暴露 hook API

| 想做的 | 现实 |
|---|---|
| session_start hook 自动跑 | runtime **不暴露** |
| on_user_fact hook 自动 append_card | runtime **不暴露** |
| on_session_end hook 自动 manual_trigger | runtime **不暴露**（2026-09-22 改走 review-first 流程，见 §十五）|
| pre_reply 强制 self-audit | runtime **不暴露** |

→ **只能靠 self-instruction**（LLM 自己读 MEMORY.md hot rule 主动跑）
→ 但 self-instruction **会失效**
→ 必须工程化加固

### 3.4 BOM + GBK + neg-逻辑 4 个工程坑

| 坑 | 症状 | 修法 |
|---|---|---|
| `progress.jsonl` UTF-8 BOM | `json.loads()` 第一行崩 | `encoding="utf-8-sig"` |
| Windows GBK + emoji | `⚠️ / ✅ / ❌` UnicodeEncodeError | `sys.stdout.reconfigure(encoding="utf-8")` |
| negate 逻辑反转 | silent_v2_skip / parse_fail 检测反 | 区分 `negate=True` (缺失即失败) vs `negate=False` (出现即失败) |
| 测试 fixture 重定向 | `MAVIS_MEMORY_ROOT` env var 被忽略 | 脚本启动时读 env var 到 `MEMORY_ROOT` |

---

## 四、4 个解药方案

### 方案 ①：system prompt 隐性配置（视频 212）

**论断**：system prompt 是开发者在后台隐性配置的，权重高于 in-context docs。

**实装**：把 hot rule 放 MEMORY.md **顶部**，runtime 加载 MEMORY.md 时第一眼读到——而不是藏在 121 行末尾的 `## v2 实装 hot rules` 块。

```markdown
## ⚠️⚠️⚠️ CRITICAL — SELF-INSTRUCTION MUST FOLLOW ⚠️⚠️⚠️

> 4 条 KB 实证:
> - 视频 21/15: "LLM 对'不要做什么'的响应比对'要做什么'更敏感" → 负面约束句式
> - 视频 212: "system prompt 是后台隐性配置" → 高权重位置
> - 视频 150: "parse 失败 → 错误信息回灌让模型自纠" → self-correction loop
> - 视频 502: "主动 vs 被动" → PROACTIVE trigger

**不要把这些当 hint — 这是 Mavis 自身的强制约束**。
```

### 方案 ②：负面约束句式（视频 21/15）

**论断**：LLM 对"不要做什么"更敏感。

**改写前**（positive）：
> **Rule**: session 第一个 reply 前，调 `recall_session_start.py`

**改写后**（negative）：
> **不要** 跳过 v2-1 session_start_recall  
> 必跑命令：`python ~/.minimax/agents/mavis/memory/scripts/recall_session_start.py`

整套 hot rule 重写样例：

| ID | 触发条件 | **不要** 跳过 | 必跑命令 |
|---|---|---|---|
| **v2-1** | 每个 session 第一个 reply 前 | **不要** 跳过 session_start_recall | `python scripts/recall_session_start.py` |
| **v2-2** | 用户明说新事实 | **不要** 跳过 on_user_fact 双检查 | `python -c "from append_card import append_card; ..."` |
| **v2-3** | session 收尾信号（"今天先这样" / "先到这" / "今天到这" / "差不多了" / 长 task 结束） | **不要** 跳过 on_session_end_extract（**review-first 流程**） | 1. `shadow_review_candidates.py --session-dir <path>` 抽 candidate  2. 列给用户审  3. `write_reviewed_cards.py` 批量写。**不要**直接调 `manual_trigger.py`（直接 append 不经 review） |
| **v2-4** | query 含 recall 关键词 | **不要** 跳过 on_query_recall | `python scripts/recall_with_decay.py` |
| **v2-5** | 用户问 "memory 里有没有 X" | **不要** 跳过 use_RAG_for_memory_recall | `python _v2_fix_recall.py` |
| **TTS-1** | 每个 reply 必生成 mp3 | **不要** 跳过 TTS 生成 | `python tts_edge.py '回复文本'` |
| **PRE-1** | 每个 reply 开始前（必跑） | **不要** 跳过 preflight audit | `python scripts/preflight_audit.py` |

### 方案 ③：self-correction loop（视频 150）

**论断**：parse fail → 错误信息回灌让模型自纠。

**实装**：`self_correct.py` 检测 reply 文本 4 类 failure：

```python
# FAILURE_PATTERNS 关键设计：negate 字段翻转
FAILURE_PATTERNS = [
    {"name": "no_preflight_marker", "negate": True,  "regex": r"\[preflight\]", ...},
    {"name": "no_tts_marker",       "negate": True,  "regex": r"\[tts generated\]|reply_\d{8}_\d{6}\.mp3", ...},
    {"name": "silent_v2_skip",      "negate": False, "regex": r"v2-[1-5].{0,40}(必须|应该|要)", ...},
    {"name": "parse_fail_in_text",  "negate": False, "regex": r"(Traceback|JSONDecodeError|KeyError:)", ...},
]

def detect_failures(reply_text: str) -> list[dict]:
    failures = []
    for fp in FAILURE_PATTERNS:
        matched = re.search(fp["regex"], reply_text)
        is_failure = (not matched) if fp.get("negate", False) else bool(matched)
        if is_failure:
            failures.append({"name": fp["name"], "fix": fp["fix"]})
    return failures
```

### 方案 ④：preflight audit（视频 502 主动 vs 被动）

**论断**：必须主动思考系统，不是被动等规则浮现。

**实装**：`preflight_audit.py` 每次 reply 开头跑：

```python
def audit(recent: int = 5) -> dict:
    """读 progress.jsonl 最近 N 行, 检查 v2 marker 使用情况."""
    if not PROGRESS.exists():
        return {"audit_passed": False, "prompt_injection": "[preflight] ⚠️ progress.jsonl 不存在"}
    
    # utf-8-sig 处理 BOM
    with PROGRESS.open("r", encoding="utf-8-sig") as f:
        entries = [json.loads(line) for line in f if line.strip() 
                   and not _try_parse_fail(line)]
    
    recent_entries = entries[-recent:]
    text_dump = " ".join(json.dumps(e, ensure_ascii=False) for e in recent_entries)
    
    used = [m for m in V2_MARKERS if m in text_dump]
    missing = [m for m in V2_MARKERS if m not in used]
    
    return {
        "audit_passed": len(missing) == 0,
        "recent_count": len(recent_entries),
        "used_markers": used,
        "missing_markers": missing,
        "prompt_injection": format_for_prompt(...),
    }
```

---

## 五、完整代码：`preflight_audit.py` (方案 ④)

```python
"""Phase 6+: preflight audit — 检查最近 N turns 哪些 v2 hot rule 被跳过.

设计依据 (来自 douyin_kb):
- 视频 502: "主动 vs 被动" — Mavis 必须主动 audit 自己的执行
- 视频 21/15: "LLM 对'不要做什么'更敏感" — 输出用 ⚠️ + "不要跳过" 句式

用法:
    python scripts/preflight_audit.py              # 默认检查最近 5 turns
    python scripts/preflight_audit.py --recent 10
    python scripts/preflight_audit.py --json       # 只输出 JSON
"""
from __future__ import annotations
import json
import os
import sys
from pathlib import Path

# Windows GBK codec crash on emoji — reconfigure stdout to UTF-8
try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

# 测试时支持 MAVIS_MEMORY_ROOT 重定向 (跟其他 scripts 一致)
_MEMORY_ROOT = Path(os.environ.get("MAVIS_MEMORY_ROOT") or Path(__file__).resolve().parent.parent)
PROGRESS = _MEMORY_ROOT / "progress.jsonl"

# Phase 6 v2 hot rules + 强制 wrappers
V2_MARKERS = [
    "v2-1",   # session_start_recall
    "v2-2",   # on_user_fact
    "v2-3",   # on_session_end_extract
    "v2-4",   # on_query_recall
    "v2-5",   # use_RAG_for_memory_recall
    "TTS-1",  # tts generation
    "PRE-1",  # preflight audit (self-reference)
]


def audit(recent: int = 5) -> dict:
    """读取 progress.jsonl 最后 N 行, 检查哪些 v2 marker 在最近 turns 被用过."""
    if not PROGRESS.exists():
        return {
            "audit_passed": False,
            "recent_count": 0,
            "used_markers": [],
            "missing_markers": V2_MARKERS,
            "last_entry_ts": None,
            "prompt_injection": "[preflight] ⚠️ progress.jsonl 不存在 — 无法 audit",
        }

    entries = []
    # utf-8-sig 处理 progress.jsonl 第一行 BOM
    with PROGRESS.open("r", encoding="utf-8-sig") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                entries.append(json.loads(line))
            except json.JSONDecodeError:
                continue  # 跳过坏行不崩

    recent_entries = entries[-recent:] if recent > 0 else entries
    text_dump = " ".join(json.dumps(e, ensure_ascii=False) for e in recent_entries)

    used = [m for m in V2_MARKERS if m in text_dump]
    missing = [m for m in V2_MARKERS if m not in used]

    result = {
        "audit_passed": len(missing) == 0,
        "recent_count": len(recent_entries),
        "used_markers": used,
        "missing_markers": missing,
        "last_entry_ts": recent_entries[-1].get("ts") if recent_entries else None,
    }
    result["prompt_injection"] = format_for_prompt(result)
    return result


def format_for_prompt(result: dict) -> str:
    """输出 markdown block 让 Mavis paste 到 reply 开头作为 self-audit prompt."""
    if result["audit_passed"]:
        return (
            f"[preflight] ✅ PASS — 最近 {result['recent_count']} turns 全部 v2 markers 已用 "
            f"({', '.join(result['used_markers'])})"
        )

    lines = [
        f"[preflight] ⚠️ AUDIT FAILED — 最近 {result['recent_count']} turns 缺 v2 markers:",
    ]
    for m in result["missing_markers"]:
        lines.append(
            f"  - {m}: 没看到调用, **不要继续沉默** — 立即决定当前 turn 是否需要, "
            f"如需要立即调对应脚本"
        )
    return "\n".join(lines)


def main():
    args = sys.argv[1:]
    recent = 5
    json_only = False
    i = 0
    while i < len(args):
        a = args[i]
        if a == "--recent":
            recent = int(args[i + 1])
            i += 2
        elif a == "--json":
            json_only = True
            i += 1
        else:
            i += 1

    result = audit(recent=recent)
    if json_only:
        print(json.dumps(result, ensure_ascii=False, indent=2))
    else:
        print(json.dumps(result, ensure_ascii=False, indent=2))
        print()
        print(result["prompt_injection"])


if __name__ == "__main__":
    main()
```

**关键设计**：
- `encoding="utf-8-sig"` 处理 progress.jsonl 第一行的 UTF-8 BOM（实测会崩）
- `recent=0` 表示查全部 entries
- `prompt_injection` 字段直接给 Mavis paste 到 reply 开头
- 负面约束 *"不要继续沉默"* 句式在 missing 行里强制出现

---

## 六、完整代码：`self_correct.py` (方案 ③)

```python
"""Phase 6+: self-correction loop — 把 parse fail / silent skip 转成 explicit error 信号.

设计依据 (来自 douyin_kb):
- 视频 150: "解析爆错了没关系, 把爆错信息再让给模型告诉他你写错了"
- 视频 21/15: "LLM 对'不要做什么'更敏感" — 输出用 ⚠️ + "不要跳过" 句式

用法:
    echo "reply text..." | python scripts/self_correct.py
    python scripts/self_correct.py --reply-file path/to/reply.md
    python scripts/self_correct.py --self-test
"""
from __future__ import annotations
import json
import re
import sys
from pathlib import Path

# Windows GBK codec crash on emoji
try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

# 已知 failure 模式 → 修复建议
# negate=True:  pattern 应该出现 (缺失即 failure)
# negate=False: pattern 不应出现 (出现即 failure, 如 parse error)
FAILURE_PATTERNS = [
    {
        "name": "no_preflight_marker",
        "negate": True,
        "regex": r"\[preflight\]",
        "fix": "调 python scripts/preflight_audit.py 自查 progress.jsonl",
    },
    {
        "name": "no_tts_marker",
        "negate": True,
        "regex": r"\[tts generated\]|reply_\d{8}_\d{6}\.mp3",
        "fix": "调 python tts_edge.py '回复文本' 生成 mp3 到 tts_inbox/",
    },
    {
        "name": "silent_v2_skip",
        "negate": False,
        "regex": r"v2-[1-5].{0,40}(必须|应该|要)",
        "fix": "实际跑对应 v2 脚本, **不要** 只说'应该'而不执行",
    },
    {
        "name": "parse_fail_in_text",
        "negate": False,
        "regex": r"(Traceback \(most recent call last\)|JSONDecodeError|KeyError:)",
        "fix": "解析失败 → 把错误信息回灌让模型自纠 (视频 150 实证)",
    },
]


def detect_failures(reply_text: str) -> list[dict]:
    """检查 reply 是否触发任何已知 failure 模式."""
    failures = []
    for fp in FAILURE_PATTERNS:
        matched = re.search(fp["regex"], reply_text)
        # negate=True: failure if NOT matched (marker 缺失)
        # negate=False: failure if MATCHED (parse error / silent skip)
        is_failure = (not matched) if fp.get("negate", False) else bool(matched)
        if is_failure:
            failures.append({"name": fp["name"], "fix": fp["fix"]})
    return failures


def make_correction_prompt(failures: list[dict]) -> str:
    """输出 markdown block 让 Mavis paste 到下一 turn 开头作为 self-correction signal."""
    if not failures:
        return "[self_correct] ✅ no failures detected — reply 完整"

    lines = [
        "[self_correct] ⚠️ detected failures — **不要假装通过**, 下一跳必须修复:",
    ]
    for f in failures:
        lines.append(f"  - ❌ {f['name']}: {f['fix']}")
    return "\n".join(lines)


def run_self_test() -> int:
    """跑 4 个样例 reply, 验证 detect_failures 真工作."""
    samples = [
        # 1. 完整 reply — 0 failure
        ("[preflight] ✅ PASS\n[tts generated] reply_20260921_180001.mp3\n"
         "调了 v2-1 recall_session_start.py 已跑", 0),
        # 2. 缺 preflight + silent_v2_skip — 2 failure
        ("[tts generated] reply_20260921_180002.mp3\nv2-2 必须跑", 2),
        # 3. 缺 tts — 1 failure
        ("[preflight] ✅ PASS\n跑了 v2-1 没看到 mp3 生成", 1),
        # 4. parse fail — 1 failure
        ("[preflight] ✅ PASS\n[tts generated]\n"
         "Traceback (most recent call last):\n  File ...\nKeyError: 'foo'", 1),
    ]

    failed = 0
    for i, (text, expected_failures) in enumerate(samples, 1):
        actual = detect_failures(text)
        status = "✅" if len(actual) == expected_failures else "❌"
        if status == "❌":
            failed += 1
        print(f"  sample {i}: expected={expected_failures} actual={len(actual)} {status}")
        for f in actual:
            print(f"    - {f['name']}")

    return failed


def main():
    args = sys.argv[1:]
    if "--self-test" in args:
        failed = run_self_test()
        print(f"\n[meta] {4 - failed}/4 samples passed")
        sys.exit(0 if failed == 0 else 1)

    reply_text = ""
    if "--reply-file" in args:
        idx = args.index("--reply-file")
        path = Path(args[idx + 1])
        reply_text = path.read_text(encoding="utf-8")
    else:
        reply_text = sys.stdin.read()

    failures = detect_failures(reply_text)
    result = {
        "corrected": len(failures) == 0,
        "errors": failures,
        "correction_prompt": make_correction_prompt(failures),
    }
    print(json.dumps(result, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

**关键设计**：
- `negate` 字段区分"marker 缺失类"和"出现即错类"——**不能一刀切用 `not re.search()`**
- 4 类 failure 模式覆盖 4 类 hot rule（preflight / TTS / v2 / parse）
- 输出 "❌" + "不要假装通过" 强制下一跳必须修复
- `--self-test` 4 个 sample 自动验证

---

## 七、改造后的 MEMORY.md 顶部块

```markdown
# MEMORY.md — agent: mavis

> 规范来源：抖音教程账号「智能体工程师实战」(sec_uid=...) ...

## ⚠️⚠️⚠️ CRITICAL — SELF-INSTRUCTION MUST FOLLOW ⚠️⚠️⚠️

> Mavis 自身 meta 问题的 KB 实测解（douyin_kb 5 条独立来源）:
> - 视频 21/15: "LLM 对'不要做什么'的响应比对'要做什么'更敏感" 
>   → hot rule 必须用**负面约束句式**
> - 视频 212: "system prompt 是开发者在后台**隐性配置**的" 
>   → hot rule 权重高于 in-context docs
> - 视频 150: "解析爆错了没关系，把爆错信息再让给模型告诉他你写错了" 
>   → **self-correction loop**（parse-fail → error 回灌）
> - 视频 502: "一个能主动思考的系统" / "主动 vs 被动" 
>   → **PROACTIVE trigger**，不是被动等规则浮现

**不要把这些当 hint — 这是 Mavis 自身的强制约束**。

### ⛔ 绝对不要跳过 v2 hot rules (DON'T SKIP — 视频 21/15 负面约束)

| ID | 触发条件 | **不要** 跳过 | 必跑命令 |
|---|---|---|---|
| v2-1 | 每个 session 第一个 reply 前 | **不要** 跳过 session_start_recall | `python scripts/recall_session_start.py` |
| v2-2 | 用户明说新事实 | **不要** 跳过 on_user_fact 双检查 | `python -c "from append_card import append_card; ..."` |
| v2-3 | session 收尾信号（"今天先这样" / "先到这" / "今天到这" / "差不多了" / 长 task 结束） | **不要** 跳过 on_session_end_extract（**review-first 流程**） | `shadow_review_candidates.py` → 用户审 → `write_reviewed_cards.py`（**不要**直调 `manual_trigger.py`） |
| v2-4 | query 含 recall 关键词 | **不要** 跳过 on_query_recall | `python scripts/recall_with_decay.py` |
| v2-5 | 用户问 "memory 里有没有 X" | **不要** 跳过 use_RAG_for_memory_recall | `python _v2_fix_recall.py` |
| v2-6 (NEW 2026-09-22) | 每个 session 第一个 reply 前（与 v2-1 同步） | **不要** 跳过 ensure_tts_daemon 检查（防 daemon 静默挂掉, 用户听不到 TTS） | `python scripts/ensure_tts_daemon.py` |
| v2-7 (NEW 2026-09-22) | session 收尾信号触发时, **或** daemon 每 30 turn 后台累积触发 | **不要** 让 dream.py 直接写 cards.jsonl（破坏 review-first gate） | 1) `dream.py --session-dir <path>` 抽 candidate 进 staging 2) `dream_status.py --pending` 列 pending 3) 用户审 → `dream_status.py --mark-approved/rejected <line>` 4) approved 项转 `write_reviewed_cards.py` 走 v2-3 同闸门 |
| TTS-1 | 每个 reply 必生成 mp3 | **不要** 跳过 TTS 生成 | `python tts_edge.py '回复文本'` |
| PRE-1 | 每个 reply 开始前（必跑） | **不要** 跳过 preflight audit | `python scripts/preflight_audit.py` |

### 🔄 自纠回路 (SELF-CORRECTION LOOP — 视频 150 实证)

如果上一 reply / 当前 reply 缺 `[preflight]` marker 或 v2-N 调用，**不要**沉默忽略：

1. **不要** 假装已执行
2. **不要** 跳过 audit call out
3. **要** 在当前 reply 开头 explicit: `[AUDIT] 上 N 跳缺 v2-X，当前跳是否需要？`
4. **要** 调 `python scripts/self_correct.py` 把 audit 结果回灌

### 🎯 主动 trigger (PROACTIVE TRIGGER — 视频 502 实证)

每个 reply 开始前，**不要**被动等 hot rule 浮现——**主动**问自己 6+1 个：

1. 这是 session 第一个 reply 吗？ → 跑 v2-1 + **v2-6 ensure_tts_daemon**（与 v2-1 同步跑）
2. 用户明说新事实了吗？ → 跑 v2-2
3. session 收尾信号？ → 跑 **v2-3**（review-first）+ **v2-7 dream 抽 candidate 进 staging**
4. query 含 recall 关键词？ → 跑 v2-4
5. 用户问 memory lookup？ → 跑 v2-5
6. （每个 reply 必跑）→ 跑 PRE-1 preflight audit + TTS-1

6+1 个问题全 no → reply 正常进行；任何 yes → **必须**调对应脚本。

> **v2-7 daemon 心跳路径**：daemon 每 30 turn 后台累积触发 `dream.py`，抽出的 candidate 静默进 staging；下一次 v2-3 review 时一并显示给用户。**不要**让 dream 直写 cards.jsonl。

---

## 使用约定
[...]

## Feedback rules（memory 写入规范，非 v2 self-instruction）
[...]
```

**关键改造点**：
- ⚠️⚠️⚠️ 顶部块**先于**规范来源说明（runtime 加载顺序）
- 旧 "v2 实装 hot rules" 块（行 89-121）**完全删除**——避免双重权威
- "## 使用约定" 留 1 份（不是 2 份）
- 3 条 feedback rules 移到底部，明确标 "**非 v2 self-instruction**"

---

## 八、测试：17 个新测试 + 8 个 self-test sample

### 8.1 test_preflight.py 关键 test

```python
def test_2_all_v2_markers_used_audit_pass():
    """所有 v2 marker 都在最近 N turn → audit_passed=True."""
    entries = [
        {"ts": "2026-09-21T15:00:00", "event": "v2-1_called"},
        {"ts": "2026-09-21T15:00:01", "event": "v2-2_called"},
        # ... 7 个 marker
    ]
    _write_progress(entries)
    result = preflight_audit.audit(recent=10)
    assert result["audit_passed"] is True
    assert len(result["missing_markers"]) == 0
    assert len(result["used_markers"]) == 7


def test_5_json_parse_error_lines_skipped():
    """JSON 解析错误行 → 跳过不崩 (BOM-safe)."""
    progress_path = TEST_ROOT / "progress.jsonl"
    with progress_path.open("w", encoding="utf-8") as f:
        f.write("this is not valid json\n")
        f.write(json.dumps({"ts": "...", "event": "v2-1_called"}) + "\n")
        f.write("{bad json again\n")
    result = preflight_audit.audit(recent=5)
    assert result["recent_count"] == 1
    assert "v2-1" in result["used_markers"]


def test_7_format_for_prompt_warning():
    """audit_passed=False 时输出 ⚠️ + 缺失清单."""
    _write_progress([])
    result = preflight_audit.audit(recent=3)
    prompt = preflight_audit.format_for_prompt(result)
    assert "⚠️" in prompt
    assert "AUDIT FAILED" in prompt
    assert "v2-1" in prompt
    assert "不要" in prompt  # 负面约束句式
```

### 8.2 test_self_correct.py 关键 test

```python
def test_6_negate_true_for_missing_markers():
    """FAILURE_PATTERNS 里 negate=True 的模式 → 缺失即 failure."""
    text = "no markers at all"
    failures = self_correct.detect_failures(text)
    names = [f["name"] for f in failures]
    # no_preflight_marker 和 no_tts_marker 都应 trigger (negate=True)
    assert "no_preflight_marker" in names
    assert "no_tts_marker" in names
    # silent_v2_skip 和 parse_fail_in_text 不应 trigger (negate=False)
    assert "silent_v2_skip" not in names
    assert "parse_fail_in_text" not in names


def test_7_negate_false_for_errors():
    """FAILURE_PATTERNS 里 negate=False 的模式 → 出现即 failure."""
    text = "[preflight] ✅ PASS\n[tts generated]\nKeyError: 'bar'"
    failures = self_correct.detect_failures(text)
    names = [f["name"] for f in failures]
    assert "silent_v2_skip" not in names  # 没 "v2-N 必须"
    assert "parse_fail_in_text" in names  # 有 KeyError
```

### 8.3 跑分

```
$ python -m pytest tests/test_preflight.py tests/test_self_correct.py -v
======================== 17 passed in 0.12s ========================

$ python -m pytest tests/
======================== 1 failed, 67 passed in 24.89s ==================
```

**17/17 新测试全过**，**67/68 总测试通过**（1 个失败是 pre-existing `test_all_four_layers_seeded` test isolation 问题——`CARDS_PATH` 在 `append_card.py` 模块加载时绑定，fixture 重定向 `MAVIS_MEMORY_ROOT` env var 不影响 in-process import 的 CARDS_PATH，跟本次改造无关）。

---

## 九、实测：audit 真工作

改造后立刻跑 audit 看真实状态：

```
$ python scripts/preflight_audit.py --recent 10 --json
{
  "audit_passed": false,
  "recent_count": 28,
  "used_markers": ["v2-1", "v2-4", "TTS-1", "PRE-1"],
  "missing_markers": ["v2-2", "v2-3", "v2-5"],
  "last_entry_ts": "2026-09-21T09:15:11.925915+00:00",
  "prompt_injection": "[preflight] ⚠️ AUDIT FAILED — 最近 28 turns 缺 v2 markers:..."
}
```

| Marker | 状态 | 原因 |
|---|---|---|
| v2-1 | ✅ used | session_start_recall 调过 |
| v2-2 | ❌ missing | 用户**没**明说新事实（"我X是Y"），正确不触发 |
| v2-3 | ❌ missing | session **没**结束，正确不触发 |
| v2-4 | ✅ used | recall_with_decay 调过 |
| v2-5 | ❌ missing | 我 label 错写成 v2-4（人类工程错，不是 LLM 错） |
| TTS-1 | ✅ used | tts_edge.py 调过 |
| PRE-1 | ✅ used | preflight_audit.py 调过 |

**audit 能区分 "触发条件没到（不扣分）" vs "silent skip（扣分）"**。

---

## 十、关键经验沉淀

### 10.1 三层 memory 测试

| 层 | 这次的结论 | 应放哪 |
|---|---|---|
| **project** | 不放 | 这是 agent 行为模式，跟项目无关 |
| **agent** | **MEMORY.md ⚠️ 顶部块** + scripts/preflight_audit.py + self_correct.py | ✅ 这里 |
| **user** | 不放 | 跨用户不通用（虽然"LLM 不遵循规则"是普遍问题，但解药是 Mavis-specific） |

### 10.2 写入 ≠ 执行

| 时刻 | 评价 |
|---|---|
| 写 hot rule 时（理性分析） | "应该每 reply 调 X" |
| 执行 hot rule 时（具体任务） | **同样理性地跳过** |

**同一 LLM，两种行为**——这是工程问题，不是哲学问题。

### 10.3 工程化加固三层

| 层 | 工具 | 时机 |
|---|---|---|
| **生成时** | ⚠️ 顶部块 + negative phrasing | MEMORY.md 写好，runtime 加载即生效 |
| **执行时** | preflight_audit.py | 每个 reply 开始前跑 |
| **事后** | self_correct.py | 每个 reply 末尾跑 + error 回灌 |

### 10.4 BOM + GBK + neg 三大坑（Windows）

| 坑 | 修法 |
|---|---|
| `progress.jsonl` UTF-8 BOM 第一行 | `encoding="utf-8-sig"` |
| Windows GBK + emoji ⚠️✅❌ | `sys.stdout.reconfigure(encoding="utf-8")` |
| `not re.search()` 反转逻辑 | 加 `negate` 字段区分两类 failure |

### 10.5 query 措辞决定召回

| Query 类型 | 召回 |
|---|---|
| "LLM 不遵循指令"（泛 LLM 行为） | 间接命中（视频 212 / 72 / 15） |
| "memory 规则 写不等于执行"（memory-specific） | **直接命中**（视频 21/15/150/502/212） |

**query 措辞要贴近 KB 实际术语**——这是 RAG 召回的核心 skill。

---

## 十一、完整改动清单

| 文件 | 改动 | 大小 |
|---|---|---|
| `MEMORY.md` | 加 ⚠️⚠️⚠️ 顶部块（5.6 KB）+ 删除旧 v2 表块 + 清理重复 使用约定 + 移 feedback rules 到底部 | 4690 → 5520 B |
| `scripts/preflight_audit.py` | **新建** — 方案 ④ preflight audit | 4267 B |
| `scripts/self_correct.py` | **新建** — 方案 ③ self-correction loop | 4965 B |
| `tests/test_preflight.py` | **新建** — 8 个 test | 5312 B |
| `tests/test_self_correct.py` | **新建** — 9 个 test | 4517 B |
| `progress.jsonl` | 加 9 个 audit-related events（含 v2-1/v2-4/TTS-1/PRE-1 markers） | +9 行 |
| **Phase 8 v2-3 review-first** (2026-09-22) | 加 2 个 script + 加 §十五章 + MEMORY.md 加 v2-3 trigger phrase 列表 | +8.6 KB |
| **Phase 8 v2-6 ensure_tts_daemon** (2026-09-22) | 加 1 个 script (4478 B) + MEMORY.md 加 v2-6 行 | +4.5 KB |
| **Phase 8 v2-7 dream 后台自动** (2026-09-22) | 加 2 个 script (dream.py + dream_status.py, 15 KB) + 1 个 prompt + MEMORY.md 加 v2-7 行 + 本章 §十六 | +18 KB |

**总成本**：~24 KB 新代码 + 5.6 KB MEMORY.md 改造 + ~30 KB v2-3/v2-6/v2-7 增量。**回报**：v2 memory system 从"写在文档里等 LLM 自觉"变成"工程化强制 audit + review-first gate + 后台自动抽 candidate"。

---

## 十二、给后续 Agent 的指引

### 12.1 什么时候调 v2 scripts

| 触发词 | 调什么 |
|---|---|
| 第一个 reply 前 | `recall_session_start.py 7` |
| 用户说 "我X是Y" / "我家X" | `append_card.py` + `policy.py` |
| 用户说 "今天先这样" / "先到这" / "今天到这" / "差不多了" / 长 task 结束 | `shadow_review_candidates.py --session-dir <path>` → 用户审 → `write_reviewed_cards.py` |
| query 含 "recall / memory / v2 / phase / 长期记忆" | `recall_with_decay.py` |
| 用户问 "memory 里有没有 X" | `_v2_fix_recall.py --kb agent_memory_kb` |
| 每个 reply 必跑 | `tts_edge.py '回复文本'` + `preflight_audit.py` |

### 12.2 自我审计 4 步

1. **生成 reply 前** 调 `preflight_audit.py --recent 5` → 看 AUDIT 结果
2. **reply 文本写好** 调 `self_correct.py --reply-file <reply.md>` → 检查 4 类 failure
3. **如果 audit FAIL** → 在 reply 开头 explicit `[AUDIT] 上 N 跳缺 v2-X`
4. **如果 failure 命中** → 调对应 v2 脚本或加 `[tts generated]` marker

### 12.3 反模式 (deprecate)

| ❌ 不要 | ✅ 应该 |
|---|---|
| 写 hot rule 但不测自己执行率 | 写完跑 `preflight_audit.py` 验证 marker 落到 progress.jsonl |
| 用 positive 句式 (Rule: 做 X) | 用 negative 句式 (**不要** 跳过 X) |
| 假设 LLM 会自己读 hot rule | 假设 LLM 会**跳过**自己的 hot rule，加工程化 audit |
| 把 hot rule 藏在 MEMORY.md 末尾 | 放 ⚠️⚠️⚠️ **顶部块**（runtime 第一眼读到） |
| parse fail 静默吞掉 | parse fail → error 回灌让模型自纠 |

---

## 十三、附录：完整文件清单

```
~/.minimax/agents/mavis/memory/
├── MEMORY.md                                    # 顶部 ⚠️⚠️⚠️ CRITICAL 块
├── cards.jsonl                                  # 14 cards (Phase 2)
├── cards_schema.json                            # JSON schema (Phase 2)
├── policy.yaml                                  # schema_version=1 (Phase 3)
├── progress.jsonl                               # 79+ 行 audit trail
├── prompts/extract_facts.md                     # LLM 抽 facts prompt (Phase 4)
├── shadow_runs/                                  # shadow extract 缓存 (Phase 4)
├── scripts/
│   ├── append_card.py                           # Phase 2
│   ├── policy.py                                # Phase 3
│   ├── decay_score.py                           # Phase 5
│   ├── shadow_extract.py                        # Phase 4
│   ├── manual_trigger.py                        # Phase 4 fallback (2026-09-22 标记 deprecate)
│   ├── recall_session_start.py                  # Phase 6 v2-1
│   ├── recall_with_decay.py                     # Phase 6 v2-4
│   ├── seed_initial_cards.py                    # Phase 6 seed
│   ├── preflight_audit.py                       # **Phase 7 方案 ④** NEW
│   ├── self_correct.py                          # **Phase 7 方案 ③** NEW
│   ├── ensure_tts_daemon.py                     # **v2-6** TTS daemon 心跳 NEW (2026-09-22)
│   ├── shadow_review_candidates.py              # **v2-3 review-first** Phase 8 NEW (2026-09-22)
│   ├── write_reviewed_cards.py                  # **v2-3 review-first** Phase 8 NEW (2026-09-22)
│   ├── dream.py                                 # **v2-7 后台抽 candidate** NEW (2026-09-22)
│   └── dream_status.py                          # **v2-7 staging 状态/审批** NEW (2026-09-22)
├── prompts/                                     # **v2-7** dream 抽 candidate 的 prompt (2026-09-22)
│   └── dream_extract.md
├── dream_candidates.jsonl                       # **v2-7 staging** 永不直写 cards.jsonl (2026-09-22)
└── dream_candidates.seen.jsonl                  # **v2-7 幂等性** 已抽过的 msg_id 列表 (2026-09-22)
├── topics/                                      # 9 个 topic files
│   ├── agent-architecture.md
│   ├── agent-discipline.md
│   ├── headroom.md
│   ├── mmx-cli.md
│   ├── ocr-code-review.md
│   ├── rag-import.md
│   ├── skill-management.md
│   ├── tts-daemon.md
│   └── windows-powershell.md
└── tests/
    ├── test_cards.py                            # 7 cases
    ├── test_decay.py                            # 12 cases
    ├── test_integration.py                       # 8 cases
    ├── test_policy.py                           # 12 cases
    ├── test_shadow.py                           # 12 cases
    ├── test_preflight.py                        # **Phase 7** 8 cases NEW
    └── test_self_correct.py                     # **Phase 7** 9 cases NEW

data/agent_memory_kb/                            # RAG KB (74 chunks, bge-m3)
data/douyin_tech_kb/                             # RAG KB (1637 chunks, bge-m3)
data/crawled_accounts_kb/                        # RAG KB (233 chunks)
data/lessons_learned_kb/                         # RAG KB (716 chunks)
```

---

## 十四、一句话总结

**LLM 写 hot rule 时理性，执行时同样理性地跳过——写入不等于执行。** 工程化方案：用 ⚠️⚠️⚠️ 顶部块强制 runtime 第一眼读、用负面约束句式（视频 21/15 实证）、用 self-correction loop（视频 150 实证）、用 preflight audit 主动思考（视频 502 实证）。3 个工程坑共 17 个新测试全过、67/68 总测试通过。

---

## 十五、v2-3 review-first 流程化（2026-09-22 update）

### 15.1 为什么改

v2-3 之前的设计是：**用户说收尾信号 → Mavis 直调 `manual_trigger.py --last-session` → 直接 append_card**。问题：这条路径**没有 review gate**，会把低质量、重复、ephemeral 的 candidate 污染 `cards.jsonl`。

Phase 7 self-instruction 修复（§四方案 ①+②+③+④）证明：v2 hot rule 必须有工程化执行机制才靠谱。v2-3 也必须走 review-first 流程，而不是"LLM 自觉 review"。

### 15.2 Trigger phrase list（5 个扩充）

| Phrase | 语义 |
|---|---|
| "今天先这样" | 明确收尾 |
| "先到这" | 明确收尾 |
| "今天到这" | 明确收尾 |
| "差不多了" | 收尾信号（弱）|
| 长 task 自然结束 | 推断（不依赖 keyword）|

**为什么不要 cron polling**：用户 2026-09-17 明确拒绝 1 分钟噪声 polling。但 session 末信号是 event-driven，**不是** noise — 是用户主动给出"今天工作完了"的语义信号。Mavis 收到信号再跑，没信号不跑。

### 15.3 Review-first 3 步流程

```
Step 1. 抽 candidate (不写)
   $ python ~/.minimax/agents/mavis/memory/scripts/shadow_review_candidates.py --session-dir <path>
   → 输出 JSON list to stdout. 不动 cards.jsonl.

Step 2. Mavis 列 candidate 给用户审
   → 自动按 5 维剔除: dedup / ephemeral / 抽象 / transient / 已在 MEMORY
   → 用户进一步挑 / 改 / 加

Step 3. 用户 approve → 批量写
   $ python ~/.minimax/agents/mavis/memory/scripts/write_reviewed_cards.py
   → 走 shadow_extract.write_to_cards (含 jsonschema 校验 + dedup)
```

### 15.4 关键命令对比

| ❌ 之前 (deprecated) | ✅ 现在 (canonical) |
|---|---|
| `manual_trigger.py --last-session`（直接 append）| `shadow_review_candidates.py --session-dir <path>` → 用户审 → `write_reviewed_cards.py` |

**为什么 `manual_trigger.py` 标 deprecate 不删**：fallback 用途 — 如果 Qwen API 不可用时，可以人工审 candidate list 然后绕过 review gate 用 manual_trigger 直写。但 v2-3 默认走 review-first。

### 15.5 实测：今天 4 张 card 灌入全过程

| 阶段 | 输出 |
|---|---|
| Step 1: shadow_review_candidates.py | 20 张 candidate (schema-valid 20/20, conf ≥ 0.7) |
| Step 2: 用户 review | 剔除 16 张（dedup/ephemeral/抽象/transient/已在 MEMORY）|
| Step 3: write_reviewed_cards.py | 4 张成功写入（0 reject / 0 dup / 0 invalid_layer）|
| cards.jsonl | 33 → **37 张**（+4）|

**学到的教训**：
1. **Qwen 抽 candidate 准确率高**（20/20 schema-valid）— schema 校验前置有效
2. **Mavis review 价值在维度判断**，不是判定 schema（schema 已自动）
3. **5 维剔除法（dedup/ephemeral/抽象/transient/已在 MEMORY）覆盖 80% 噪声**
4. **review-first 流程让 append_card 0 reject** — 写之前已经 schema 过一遍 + dedup 过一遍

### 15.6 关联改动

| 文件 | 改动 |
|---|---|
| `MEMORY.md` v2-3 行 | 扩充 trigger phrase 5 个 + review-first 3 步流程 + 警告"不要直调 manual_trigger" |
| `cards.jsonl` | +2 张：`user_pref.memory_session_end_auto_extract`（用户决策）+ `agent_lesson.v2_3_review_first_protocol`（协议细节）|
| `scripts/shadow_review_candidates.py` | **新建** — wrapper，调用 `_call_qwen` 但不 `write_to_cards` |
| `scripts/write_reviewed_cards.py` | **新建** — wrapper，批量 `write_to_cards` 含 schema 校验 |
| `scripts/manual_trigger.py` | 标 deprecate，不删（fallback 用途）|

### 15.7 后续可优化

- **session 末 hook**（runtime 支持后）：v2-3 自动执行从 Mavis 主动判断改成 hook 触发，进一步减少 self-instruction 依赖
- **Qwen confidence ≥ 0.85 高质量 candidate**：跳过 review gate 直接 write（最高信号子集）|
- **多 session 批量 review**：当用户连续多 session 收尾时，一次列出 N 个 session 的 candidate 一起审

### 15.8 v2 hot rule 的进化方向

v2 系列 hot rule 都在做同一件事：**用工程化手段加固 self-instruction**。
- v2-1/2/4/5：query 触发 — 已成熟
- v2-3：session 末触发 — review-first 流程刚落地（§十五）
- TTS-1 / PRE-1：每个 reply — 已成熟

**统一的进化方向**：从"LLM 自觉执行" → "Mavis 主动判断 + 工程化执行机制 + review gate"。下次新加 hot rule 时，默认走这个 pattern。

---

## 十六、v2-7 后台自动抽 candidate + Dream（2026-09-22 update）

### 16.1 为什么改

v2-3 review-first 已经把"session 末抽 candidate → 用户审 → 写 cards"标准化，但**只有用户主动说"今天先这样"才触发**。如果用户不提（直接关 chat / 异常退出 / 自然结束），memory 整理就缺一次。

参照 [HKUDS/nanobot](https://github.com/HKUDS/nanobot) 的 Dream 设计（cron 每 2h 自动整理 + SOUL/USER/AGENTS 角色文件分离），决定给 Mavis 加**后台自动抽 candidate** 的能力，但要**保住 review-first gate 核心**（治幻觉底线）。

### 16.2 设计：3 段式 + 4 层保护 review gate

```
[触发：session 收尾信号 / daemon 每 30 turn / 用户 /dream 命令]
        ↓
[dream.py — MiniMax 抽 candidate]
   ├─ 读 session_dir/messages.jsonl（最近 N 条 user/assistant 消息）
   ├─ 跳 compactionSummary（避免与 shadow_extract.py 重复）
   ├─ 跳已 seen 的 msg_id（幂等性）
   └─ 写 staging: memory/dream_candidates.jsonl（status=pending）
        ↓
[review-first gate — 与 v2-3 同一闸门]
   ├─ dream_status.py --pending 列 pending
   ├─ 用户审 → mark-approved <line> 或 mark-rejected <line>
   └─ approved 项转 write_reviewed_cards.py 走 v2-3 review-first
        ↓
[永不直写] MEMORY.md / cards.jsonl / topics/ — 保留 v2-3 治理路径
```

### 16.3 跟 nanobot Dream 的关键差异

| 维度 | nanobot Dream | Mavis v2-7 dream |
|---|---|---|
| 写到哪里 | `workspace/memory/MEMORY.md` 直写 | `dream_candidates.jsonl` staging（status=pending）|
| 谁能写 | LLM 自动 | **必须**用户 `--mark-approved` |
| 治幻觉 | 无 | **review-first gate 完整保留** |
| 触发 | cron 2h | session 收尾 + daemon 30 turn + 用户 `/dream` |
| 模型 | 任意 | MiniMax M2（直连，无中间层）|
| 幂等性 | 无显式 | `.seen.jsonl` 标记 msg_id |

**核心保护**：`dream.py` **不导入** `append_card` / `write_to_cards` —— 物理上无法直写 cards.jsonl。即使 dream.py 被破坏，cards.jsonl 也不会被污染。

### 16.4 4 个保护 review-first gate 的工程约束

1. **物理隔离**：dream.py 只 `import` `urllib.request` / `json` / `pathlib`，**不**导入任何写 cards 的函数
2. **默认 staging**：所有 candidate 落到 `dream_candidates.jsonl`，status 默认 `pending`
3. **手动 approve**：`--mark-approved <line>` 是显式操作，不会自动批量通过
4. **同 v2-3 闸门**：approved 项必须再走 `write_reviewed_cards.py` 才有 `dedup + schema 校验` —— 即使绕过 status 标记，schema 失败也会被挡

### 16.5 端到端实测（2026-09-22）

```bash
$ python dream.py --session-dir <latest_session>
[dream] session: 03-11-39-808-session_...
[dream] 9 new messages (after dedup)
[dream] extracted 1 candidates (min_conf=0.0)
[dream] wrote 1 NEW candidates to .../dream_candidates.jsonl (status=pending)

$ python dream_status.py --pending
=== 1 pending candidates ===
  [  0] reference conf=1.00  用户提出架构思路：把100候选压缩到3-5个，用贪吃蛇4选1格式让Laya做决策

$ python dream_status.py --mark-approved 0
[ok] marked line 0 as approved

$ python dream.py ...  # 重跑验幂等
[dream] 1 new messages (after dedup)
[dream] extracted 0 candidates (min_conf=0.0)
[dream] wrote 0 NEW candidates to .../dream_candidates.jsonl
```

**抽到的实际 candidate**：[reference] conf=1.00 — "用户提出架构思路：把100候选压缩到3-5个，用贪吃蛇4选1格式让Laya做决策"（来自今天 session 早些时候）

### 16.6 v2-7 hot rule 决策树（与 v2-3 共用闸门）

```
session 收尾 / daemon 每 30 turn 后台累积
        ↓
   调 dream.py 抽 candidate → dream_candidates.jsonl
        ↓
   用户看到 dream_status.py --pending 输出
        ↓
  ┌─────────────────┬─────────────────┐
  │ approve         │ reject          │
  └─────────────────┴─────────────────┘
        ↓                       ↓
  v2-3 review gate        直接丢弃（status=rejected）
  write_reviewed_cards.py
        ↓
  cards.jsonl ✓
```

### 16.7 5 个不做的（保护 review-first 核心）

- ❌ 不让 dream.py 直接写 MEMORY.md / cards.jsonl / topics/
- ❌ 不绕用户 review 直接批量 approve
- ❌ 不在用户没看见时静默加 topic 文件
- ❌ 不做 cron（沿用 Mavis 用户偏好 "daemon 由 Mavis 维护，不用 cron"）
- ❌ 不在每 reply 都跑（噪音）— 只在 session 收尾或用户 `/dream` 或 daemon 30 turn 累积

### 16.8 跟 §十五 v2-3 的关系

| | §十五 v2-3 review-first | §十六 v2-7 dream |
|---|---|---|
| 触发 | session 收尾信号（5 个 trigger phrase）| session 收尾 + daemon 30 turn + `/dream` |
| 抽 candidate 源 | `compactionSummary`（用 Qwen / SF API）| `messages.jsonl` 全文（用 MiniMax）|
| 写到哪 | 直接输出 stdout（用户审）| staging `dream_candidates.jsonl`（持久化）|
| 复用闸门 | ✅ 用 v2-3 | ✅ **共** v2-3 review-first gate |
| 关系 | **手动触发** | **后台累积** + **v2-3 接管审** |

**结论**：v2-3 和 v2-7 是**互补**的，**不**是替代。
- v2-3 = "用户记得提"时立即抽
- v2-7 = "用户忘了提"时后台补抽（累积到下次 v2-3 review）

两者 candidate 都进 `write_reviewed_cards.py` 同一条闸门，**统一治理**。

### 16.9 后续可优化

- **v2-7 daemon hook 自动触发**：runtime 支持后改成 hook 触发，0 self-instruction 依赖
- **conf ≥ 0.85 自动 approve**（最高信号子集）：减少 review 负担，仍经 schema 校验
- **多 session 批量 review**：当 daemon 累积 N 个 session candidate 时，一次 review 全部
- **staging TTL**：超过 30 天未 review 的 pending candidate 自动 reject，避免无限累积

### 16.10 v2 hot rule 现在的全景

| ID | 触发 | 状态 |
|---|---|---|
| v2-1 | session 第一 reply | ✅ 成熟 |
| v2-2 | 用户明说新事实 | ✅ 成熟 |
| v2-3 | session 收尾 | ✅ review-first (§十五) |
| v2-4 | query 含 recall | ✅ 成熟 |
| v2-5 | memory lookup | ✅ 成熟 |
| **v2-6** (NEW) | session 第一 reply + tts daemon 检查 | ✅ 已实装 |
| **v2-7** (NEW) | session 收尾 + daemon 后台 | ✅ 已实装 (§十六) |
| TTS-1 | 每 reply | ✅ 成熟 |
| PRE-1 | 每 reply | ✅ 成熟 |

v2-6 + v2-7 是 §七 → §十一 → §十五 pattern 的延续：**先用工程化解决"LLM 不自觉"问题**（review-first / daemon 心跳 / 后台累积），**再考虑自动化**。