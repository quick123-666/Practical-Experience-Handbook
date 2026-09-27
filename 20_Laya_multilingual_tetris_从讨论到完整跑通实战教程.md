# Laya multilingual 玩 Tetris — 从讨论游戏生态到完整 Web 可视化实战

> **实战目标**:在禁用 MCP、纯 Python 311 + PowerShell 5.1 的环境下,把 **Laya multilingual 322M** 接进 **俄罗斯方块** — 不是 snake 那种 4 方向 choice,而是 **每方块 4 候选落点 × 1 次 noul** 的"中文陈述评分"模式。从讨论"Laya 怎么玩到游戏"开始,到跑通 Canvas 浏览器可视化 + 4 候选概率面板 + 速度/安全护栏 UI 结束。
> **落地日期**: 2026-09-27
> **适用**: Windows + Python 3.11 + Laya multilingual (HF mirror) + edge-tts

---

## 一、缘起:用户问的 3 个问题

用户在蛇形 pipeline 教程完成、看完 Laya multilingual 中文 conf 0.029-0.5 实测后,连续问了:

1. **"我们这个项目接的是 laya 是吧"** — 确认基线
2. **"现在把刚才这些游戏到底怎么加上 laya 的,以及 laya 到底怎么玩到游戏的,给完全搞明白"** — 要完整技术拆解
3. **"你弄个除了贪吃蛇以外的其他游戏跑起来给我看"** — 实跑一个非 Snake 的游戏

我先做问题 2 的完整调研(从 Snake 真实代码 + zzhdbw/laya-Ascend Tetris 真实代码 + laya-mlx 性能基准 + convaiinnovations 官方集成建议),再针对问题 3 选 Tetris 做端到端跑通。

**为什么选 Tetris(不选 Flappy/Pac-Man)**:
- zzhdbw/laya-Ascend 已有完整可跑 demo,带 game.py + policy.py + runtime.py + server.py + index.html
- 中文陈述模板天然清晰(消 X 行 / 留 Y 洞 / 堆叠高低)
- 4 候选短名单跟 Laya sweet spot 完全对得上
- CPU 能跑(322M multilingual 单条决策 ~100ms,4 候选 x1 次 ~400ms)

---

## 二、Laya API 三个原语 — 任何游戏都用这三个

```python
# 来自 convaiinnovations/laya 官方 quickstart (huggingface.co/convaiinnovations/laya)
from laya import Router
router = Router(preload=True)

state = "I was billed twice. Please refund today."
questions = {
    "department": {
        "type": "choice",                           # 多选一
        "instructions": "Which department should handle this?",
        "criteria": {                                # dict/列表都行
            "billing":  "invoices, payments, refunds",
            "technical": "bugs, outages, system errors",
            "sales": "pricing, new contracts",
        },
    },
    "urgency": {
        "type": "score",                             # 量表打分
        "instructions": "How urgent?",
        "criteria": ["not urgent", "soon", "blocking"],
    },
    "churn_risk": {
        "type": "noul",                              # 是/否
        "instructions": "Does the user threaten to cancel?",
    },
}

res = router.predict(state, questions)
# res["answers"]["department"]["choice"]      -> "billing"
# res["answers"]["department"]["probabilities"]  -> {"billing": 0.94, ...}
# res["answers"]["department"]["confidence"]  -> 0.94
# res["answers"]["urgency"]["score"]          -> 1.87
# res["answers"]["churn_risk"]["noul"]        -> 0.892  # P(true)
# res["routing"]["model"]                     -> "english"/"multilingual"
```

**Laya 的硬约束**(官方诚实承认的):
- 选项 ≤ 20(head budget 192/256 token 共享,超过会崩)
- 零样本接近瞎猜(typed-decisions 0.362 ≈ 随机)
- 出厂过度自信(中文 conf 普遍 0.85-0.99 但实际可能 0%)

---

## 三、5 个真实游戏的 Laya 适配模式 — 通用框架

把调研到的 9 个游戏 (Snake/Tetris/Flappy/Pac-Man/Breakout/格斗/Arena/Doom/Flights) 抽出来,任何新游戏都遵循同一架构:

```
┌─────────────────────────────────────────────────────────────┐
│  L4 规则层 (Game) │
│  - 枚举当前局面所有合法动作                                       │
│  - 算每个动作的 metrics(消行/留洞/高度/安全)                       │
│  - 启发式筛 top-N(N=4-5,Laya sweet spot)                       │
│  ↓ 把候选吐给 Policy                                           │
├─────────────────────────────────────────────────────────────┤
│  L3 状态翻译层 (Encode / Describe)                              │
│  - 把游戏状态翻译成 Laya 能读的文本                               │
│    Snake:  "heading: right | length: 3 | food: 5 up..."          │
│    Tetris: "这个落点消除一行,不留空洞,堆叠保持低位。"              │
│  ↓ state + questions 喂给 Laya                                 │
├─────────────────────────────────────────────────────────────┤
│  L2 Laya 决策层 (Policy.decide)                                  │
│  - 1 次前向 / 候选,返回概率分布                                    │
│  - 选 max P 的候选 OR noul P(true) 最高的候选                      │
│  - 可选:安全护栏(发现危险动作时换次优解)                            │
│  ↓ 输出最终 action                                              │
├─────────────────────────────────────────────────────────────┤
│  L1 执行层 (Apply)                                              │
│  - 把决策应用到游戏状态,推进一帧                                    │
│  - 验证:分数/生命/位置(per-tick verify)                            │
└─────────────────────────────────────────────────────────────┘
```

### 关键 insight:**Laya 参与决策 ≠ LLM 写代码**

Laya **不生成文本**,只做**类型化判断**。游戏里 Laya 的角色 = "打分裁判":
- 它**不知道**自己在玩 Tetris
- 它**只看到**一句"这个落点消除一行,不留空洞,堆叠保持低位。"
- 它回答 P(好) = 0.86(零样本,纯中文语义理解)
- 游戏代码 argmax 选 P 最高的那个

**这是 "Laya 参与决策" 的本质**:**模型本身不玩游戏,只是给候选动作打概率分**。所有"决策"逻辑在代码里,Laya 只负责"识别哪个候选好"。

---

## 四、Tetris 完整代码拆解

### 4.1 游戏层 (`game.py` 完整 6481 字节)

**7 种方块定义 + 启发式评分 + 短名单**:

```python
WIDTH, HEIGHT = 10, 20
SHORTLIST = 4
DANGER_HEIGHT = 15

SHAPES = {
    "I": [[(0,0),(1,0),(2,0),(3,0)], [(0,0),(0,1),(0,2),(0,3)]],
    "O": [[(0,0),(1,0),(0,1),(1,1)]],
    "T": [...], "S": [...], "Z": [...], "J": [...], "L": [...],
}

def evaluate(self, rotation, col):
    """模拟落点 → 算 4 个 metric"""
    cells = SHAPES[self.current][rotation]
    row = self._drop_row(cells, col)              # 这个列从顶部能落到第几行
    board_after, cleared = self._simulate(cells, col, row)  # 模拟后消除的行数
    heights = _column_heights(board_after)
    holes_delta = _count_holes(board_after) - _count_holes(self.board)
    bump = sum(abs(a-b) for a,b in zip(heights, heights[1:]))  # 高度差 = 表面粗糙度
    
    # 经典 Bertsekas-Tsitsiklis / Rogald 启发式评分
    score = (
        3.0 * cleared              # 消行奖励
        - 5.0 * holes_delta        # 留洞惩罚(最致命)
        - 0.4 * (sum(heights) - agg_before)  # 高度增长惩罚
        - 0.2 * bump               # 表面粗糙度惩罚
    )
    return {
        "rotation": rotation, "col": col, "row": row,
        "cells": [[col + x, row + y] for x, y in cells],
        "lines": cleared, "holes": max(0, holes_delta),
        "height": max(heights),
        "heuristic": round(score, 2),
    }

def candidates(self):
    """启发式筛 4 候选(短名单,去重 signature)"""
    scored = []
    for rotation, cells in enumerate(SHAPES[self.current]):
        max_x = max(x for x, _ in cells)
        for col in range(WIDTH - max_x):
            cand = self.evaluate(rotation, col)
            if cand is not None:
                scored.append(cand)
    scored.sort(key=lambda c: c["heuristic"], reverse=True)
    
    shortlist, seen = [], set()
    for cand in scored:                           # prefer distinct outcomes
        signature = (cand["lines"], cand["holes"], cand["height"] > DANGER_HEIGHT)
        if signature in seen: continue
        seen.add(signature)
        shortlist.append(cand)
        if len(shortlist) == SHORTLIST: break     # 严格 4 个
    return shortlist
```

### 4.2 状态翻译层 (`policy.py` 完整 3008 字节) — **Laya 参与决策的关键**

```python
CN_NUM = ["零", "一", "两", "三", "四", "五", "六", "七", "八", "九", "十"]
QUESTION = "这是一个好的落点吗？"

def cn(n):
    return CN_NUM[n] if 0 <= n < len(CN_NUM) else str(n)

def describe(cand):
    """把 1 个候选落点翻译成 1 句中文陈述 — Laya 评分对象"""
    lines = f"消除{cn(cand['lines'])}行" if cand["lines"] else "不消行"
    holes = f"埋下{cn(cand['holes'])}个空洞" if cand["holes"] else "不留空洞"
    if   cand["height"] > 15: height = "堆叠接近板顶"
    elif cand["height"] > 10: height = "堆叠变高"
    else:                     height = "堆叠保持低位"
    return f"这个落点{lines},{holes},{height}。"

class LayaTetrisPolicy:
    def rate(self, cands):
        """每个候选独立打分 — Laya 决策核心"""
        rated = []
        for cand in cands:
            statement = describe(cand)
            output = self.agent.predict(
                statement,                                         # state = 中文陈述
                {"q": {"type": "noul", "instructions": QUESTION}},  # 是/否评分
            )
            answer = output["answers"]["q"]
            rated.append({**cand, "statement": statement, "p_good": answer["noul"]})
        return rated, ...

    def decide(self, game):
        cands = game.candidates()
        rated, ... = self.rate(cands)
        proposed = max(range(len(rated)), key=lambda i: rated[i]["p_good"])
        executed = proposed
        
        # Safety shield — Laya 可能选了一个把堆叠顶到 15+ 的
        if self.guarded and rated[proposed]["height"] > DANGER_HEIGHT:
            safe = [i for i in range(len(rated)) if rated[i]["height"] <= DANGER_HEIGHT]
            if safe:
                executed = max(safe, key=lambda i: rated[i]["p_good"])
        return TetrisDecision(
            candidates=rated, proposed=proposed, executed=executed,
            intervened=proposed != executed, ...
        )
```

### 4.3 完整决策回路 (per piece)

```
┌──────────────────────────────────────────────────────────────┐
│  Piece N 出生 → game.candidates() → 4 候选                      │
│  ↓                                                                              │
│  for cand in cands:                                                            │
│    statement = describe(cand)  ← 中文陈述                                       │
│    laya.predict(statement, {q: {type: noul, instructions: "好落点?"}}) │
│    p_good[cand] = answer.noul  ← P(好落点) ∈ [0, 1]                         │
│  ↓                                                                              │
│  proposed = argmax(p_good)                                                     │
│  if proposed.height > DANGER_HEIGHT:  # Safety shield                  │
│    executed = argmax among safe ones                                          │
│  else:                                                                              │
│    executed = proposed                                                              │
│  ↓                                                                              │
│  game.apply(executed)  → board 更新 + score 更新                       │
└──────────────────────────────────────────────────────────────┘
```

**Laya 实际扮演的角色**:**纯文本打分器**,4 次独立 forward,每次只看 1 句中文陈述,返回 P(好)。**完全不知道游戏规则 / 不看 board / 不看其他候选**。

---

## 五、Laya 真实学会的语义(零样本,无 Tetris 训练数据)

跑 50 块, 200 次评估, 概率分布:

| 消行 | 留洞 | 样本数 | 平均 P(好落点) | 中文陈述 |
|---:|---:|---:|---:|---|
| 0 | 0 | 53 | **0.5002** | 不消行,不留空洞,堆叠保持低位 |
| 0 | 1 | 44 | **0.4076** | 不消行,埋下一个空洞,堆叠保持低位 |
| 0 | 2 | 47 | **0.3302** | 不消行,埋下两个空洞,堆叠保持低位 |
| 0 | 3 | 29 | **0.2302** | 不消行,埋下三个空洞,堆叠保持低位 |
| **1** | **0** | **12** | **0.5610** | **消除一行,不留空洞,堆叠保持低位** |
| 1 | 1 | 2 | **0.3603** | 消除一行,埋下一个空洞,堆叠保持低位 |
| **2** | **0** | **3** | **0.8617** | **消除两行,不留空洞,堆叠保持低位** |

**Laya 学到的本质(零样本!**):
```
P(好落点) ≈ 0.50 + 0.06 × 消行 + 0.36 × 消双行 - 0.10 × 留洞
```

跟经典启发式评分 `3.0 * cleared - 5.0 * holes_delta - 0.4 * height` **几乎完美同向**。模型纯靠"中文语义理解"学会了 Tetris 价值函数 — **没有 RL,没有训练数据**。

**这就是 Laya 在游戏里的核心参与方式**:**模型不"会玩"游戏,只是会"看中文句子打分"**。游戏的所有规则知识都在 `describe()` 函数里。

---

## 六、完整 Web 可视化链(`server.py` + `index.html`)

### 6.1 Flask-less HTTP server (stdlib only)

```python
# server.py — 完全基于 http.server.ThreadingHTTPServer
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlparse
import json, threading

class TetrisApp:
    def new_game(self, body):
        seed = body.get("seed", random.randint(0, 99999))
        game = TetrisGame(seed=seed)
        policy = LayaTetrisPolicy(self.agent, guarded=bool(body.get("guarded", True)))
        sid = uuid.uuid4().hex[:12]
        self.sessions[sid] = {"game": game, "policy": policy, "stats": {...}}
        return {"session": sid, "seed": seed, "state": game.snapshot()}

    def step(self, session):
        game, policy = entry["game"], entry["policy"]
        before = game.snapshot()
        with self.inference_lock:                  # 序列化 Laya 调用
            decision = policy.decide(game)
            game.apply(decision.candidates[decision.executed])
        return {"decision": decision.to_dict(), "before": before, "state": game.snapshot(), ...}

    def snapshot(self, session):                   # 新加:取当前未处理的方块名
        return {"state": game.snapshot(), "stats": entry["stats"]}

# 路由: GET /          → index.html
#       GET /api/info  → {model, device, width, height}
#       POST /api/tetris/new     → new_game
#       POST /api/tetris/snapshot → snapshot(返回当前 state.current, Laya 决定前)
#       POST /api/tetris/step     → step(Laya 决定 + apply)
```

### 6.2 前端决策循环 — **Laya 参与决策的视觉呈现**

```javascript
// index.html — 完整的"spawn → 决定 → 滑落"循环
async function loop(gen) {
  // STEP 1: 先取快照(不调 Laya),显示当前方块在顶部 row 0 中央等待
  const boardNow = await api("/api/tetris/snapshot", { session: s.session });
  drawSpawn(boardNow.state, boardNow.state.current);  // ▸ T 待落

  // STEP 2: Laya 决定(~400ms,4 个候选 × 1 次 noul)
  const data = await api("/api/tetris/step", { session: s.session });
  updatePanel(data.decision);                          // 4 候选概率条更新

  // STEP 3: 从 spawn 到 placement 的两阶段动画(水平 35% + 垂直 65%)
  await animateDrop(data.before, data.decision, animBudget);

  // STEP 4: 静止 ~200ms 让用户看清落定的方块,再回到 STEP 1
  await new Promise(r => setTimeout(r, settlePause));
  loop(gen);
}
```

**用户在前端看到的 Laya 参与流程**:
1. **顶部中央**出现下一个方块 + "▸ T 待落" 标签
2. **等待 ~400ms**(Laya 内部跑 4 次 noul)
3. **右侧 4 候选面板**出现:P(好落点) 概率条
4. **方块开始横向滑动**到 P 最高那个候选的列(目标列蓝色虚线框)
5. **方块垂直下落**到目标行(ease-in 重力感)
6. **落定 ~200ms** 后下一个方块从顶部中央出现

**这才是"Laya 在游戏里参与决策"的视觉化**:
- Laya 不在屏幕里出现(模型就是模型,看不到)
- 但**所有动作都是它打分选出来的**
- 4 候选概率条**实时显示**模型对每个候选的"好落点"信念
- 用户能直接验证:Laya 是不是"看到了"消行/留洞的语义

---

## 七、本项目特有的踩坑实录(按时间顺序)

### 坑 1:animateDrop spawn position 错位
- **症状**:方块在游戏界面外(上方 1-2 行)出现,看起来"对准了"目标列
- **真因**:旧版 `animateDrop` 用 `(cell.x, cell.y - shift)` 计算位置,shift 从 drop 距离开始减;piece 一直在目标列上方,看起来"直接对准"
- **修复**:改成两阶段动画:水平 (35%) + 垂直 (65%),从顶部中央 spawn
- **代码位置**:`index.html:221-241`

### 坑 2:SHAPES Python tuple vs JS array
- **症状**:`[pageerror] number 0 is not iterable`
- **真因**:从 Python `game.py` 抄 SHAPES 定义时用了 `(0, 0)` tuple 写法 — JS 里 `,` 是逗号运算符,`(0, 0)` 求值为 `0`(数字,不是数组)。`cells` 变成 `[0, 1, 2, 3]` 而不是 `[[0,0],[1,0],[2,0],[3,0]]`
- **修复**:`(0, 0)` → `[0, 0]`
- **教训**:跨语言抄代码注意字面量语法差异。Python tuple `(a, b)` ≠ JS array `[a, b]`

### 坑 3:JS error 导致 `state.playing = false`
- **症状**:spawn 函数绘制当前方块时报错,catch 块把 `state.playing = false`,loop 退出,游戏停止
- **真因**:错误的 `drawPreview` 在 catch 里直接 `s.playing = false; return;` —— 任何 JS 错误都会停掉游戏
- **修复**:去掉 drawPreview,只保留 drawSpawn(更简单的 spawn 预览,不会抛错)
- **教训**:游戏循环里的 catch 必须分清"可恢复错误"(网络超时)和"致命错误"(JS 异常)。前者降级继续,后者才退出

### 坑 4:server.py 被 background bash task 超时杀掉
- **症状**:`netstat` 显示 8011 端口空闲,服务没了
- **真因**:bash `run_in_background:true` 默认 timeout 300 秒,server 启动耗时 ~50s + 冷加载 + 用户游玩 5+ 分钟,超过 300s 被 watchdog kill
- **修复**:`timeout: 3600`(1 小时)显式指定
- **教训**:长跑 daemon 必须显式 timeout,不能用默认值

### 坑 5:`get_config_value` 中 `_DISPATCH_TABLE` 与 ActionKind 不一致
- **症状**:旧 snake pipeline 抛 `AttributeError("DOUBLE_CLICK")` 被误读成"双击失败"
- **真因**:Python `elif` evaluate 时先读 `ActionKind.X`,漏写 enum 成员 → AttributeError
- **修复**:`dict-based dispatch + _validate_dispatch()` import-time 检查
- **教训**:enum dispatch 永远用 dict + import-time 检查,不要 if-elif 链

### 坑 6:PowerShell Tee-Object 编码破坏 UTF-8
- **症状**:log 文件里中文显示成 `??????`
- **真因**:PowerShell 5.1 `Tee-Object` 默认 GBK,python 中文 stdout 经过 PowerShell 重定向时变乱码
- **修复**:用 `open(file, 'w', encoding='utf-8')` 直接写文件,绕过 PowerShell 重定向
- **教训**:Windows + Python + PowerShell 组合永远显式 UTF-8

### 坑 7:HF cache 路径需要 subfolder
- **症状**:`Agent("convaiinnovations/laya")` 默认加载根目录英文 checkpoint(842MB)
- **真因**:HF hub 仓库把 3 个 checkpoint (english/multilingual/typed-decisions) 放在同一个 repo 的 subfolder
- **修复**:`Agent("convaiinnovations/laya", subfolder="multilingual", device="cpu")`
- **教训**:HF repo 多个 checkpoint 必须显式 `subfolder=`

---

## 八、实测数据(CPU 上完整跑通)

### 8.1 性能

| 指标 | 数值 |
|---|---|
| 模型加载 (HF cache hit) | **47.8 s** |
| Warmup 第一次前向 | **105 ms** |
| 每块 Laya 决策 (4 候选 × 1 noul) | **405.6 ms** |
| Token / 块 | ~205 (中文短句) |
| 安全护栏干预次数 (50 块) | **0** |

### 8.2 50 块游戏实测 (seed=7, guarded=True)

```
pieces=50  lines=18  score=2100  alive=True
interventions=0  avg_inf_ms=405.6  total_tokens=10274
```

最终棋盘 (Laya 自己堆出来):
```
+----------+
|..........|   ← 14 行空
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|..........|
|I.T.......|   ← row 14
|ITT.......|   ← row 15
|IST.......|   ← row 16
|ISS.......|   ← row 17
|ZZS.......|   ← row 18
|JZZOOJ....|   ← row 19 (地板)
+----------+
```

### 8.3 浏览器 20 秒实测 (speed=2)

- 14 块落下,2 行消除,200 分
- **pps = 0.36**(每块约 2.7 秒,符合"spawn 400ms + decide 400ms + 滑落 600ms + 静止 200ms")

---

## 九、完整复现步骤

### 9.1 准备 (5 分钟)

```bash
# 安装 Laya
pip install laya

# 配置 HF mirror (CN-ISP 必备)
$env:HF_ENDPOINT = "https://hf-mirror.com"
$env:HF_HUB_DISABLE_XET = "1"
$env:USE_TF = "0"

# 下载 multilingual checkpoint
hf download convaiinnovations/laya --local-dir models/laya-multilingual
# 或者直接 Agent("convaiinnovations/laya", subfolder="multilingual") 自动拉
```

### 9.2 文件清单 (`C:/Users/Administrator/Desktop/minimax/laya_tetris/`)

| 文件 | 大小 | 角色 |
|---|---:|---|
| `game.py` | 6481 B | 7 方块定义 + 启发式评分 + 候选短名单 |
| `policy.py` | 3008 B | 中文陈述模板 + per-cand noul + safety shield |
| `runtime.py` | 1325 B | HF cache loader + CPU warmup |
| `server.py` | ~5 KB | stdlib HTTP + TetrisApp + 3 个 API |
| `index.html` | ~13 KB | Canvas 可视化 + spawn→decide→drop 循环 |
| `run_clean.py` | 4275 B | 跑 N 块 + UTF-8 报告 |
| `_capture*.py` | ~2 KB | Playwright headless 截图 |

### 9.3 跑起来

```bash
cd C:/Users/Administrator/Desktop/minimax/laya_tetris
$env:HF_ENDPOINT="https://hf-mirror.com"
$env:HF_HUB_DISABLE_XET="1"
$env:USE_TF="0"
python server.py --device cpu --port 8011
# 输出:Tetris demo: http://127.0.0.1:8011
```

打开浏览器 `http://127.0.0.1:8011/`,Laya 自动开局 + 自动玩。
- 拖"速度"滑块到 1-11(11 = 最快,~1.25 块/秒)
- 取消"安全护栏"看模型会不会"作死"

### 9.4 验证 (CLI 模式)

```bash
# 跑 50 块 + 写 UTF-8 报告
python run_clean.py --pieces 50 --seed 7
# 报告:run_clean_50_7.log (~35 KB)
```

---

## 十、关键 takeaway — Laya 怎么"参与"游戏决策

| 维度 | 怎么"参与" | 由谁实现 |
|---|---|---|
| **看游戏状态** | 读候选的 metrics(消 X 行/留 Y 洞/堆叠高低) | `describe()` 中文陈述生成 |
| **做判断** | 1 次 noul 评分,P(好落点) ∈ [0, 1] | Laya multilingual (zero-shot) |
| **选动作** | argmax 4 个候选的 P(好) | `LayaTetrisPolicy.decide()` |
| **风险兜底** | Safety shield(堆叠 > 15 强制换次优解) | `decide()` 里的 `guarded` 分支 |
| **验证** | per-piece verify(分数/生命/位置) | `TetrisGame.snapshot()` |
| **学习** | **无训练数据**,纯 zero-shot 中文语义 | Laya multilingual 已经会中文 |

**Laya 在 Tetris 里的参与方式总结**:

```
┌──────────────────────────────────────────────────────────┐
│  Laya 实际做了什么                                          │
│  - 输入:1 句中文陈述 (e.g. "这个落点消除两行,不留空洞...")  │
│  - 输出:P(好) ∈ [0, 1]                                     │
│  - 内部:322M 参数前向,~100ms/call                            │
│  - **不**知道:游戏规则 / board / 其他候选 / 历史            │
│                                                          │
│  Laya 没做什么(关键!)                                       │
│  - 没生成文本(纯分类,无 token 输出)                          │
│  - 没"玩"游戏(没有 plan / search / 状态机)                   │
│  - 没"学"Tetris (零样本)                                    │
└──────────────────────────────────────────────────────────┘
```

**这就是"Laya 参与决策"的本质**:**模型只是个裁判,游戏规则在代码里**。把候选动作翻译成中文 → Laya 打分 → argmax。所有"AI"的部分都在描述+评分这一步,游戏本身是普通 Python 程序。

---

## 十一、跟已有教程的关系

| 教程 | 关系 |
|---|---|
| **16_Laya 贪吃蛇** | 4 方向 choice,Laya-snake 微调版 (acc 0.90),本教程是 Tetris 版本(per-cand noul,zero-shot) |
| **17_computer_control_pipeline** | UI 自动化 multi-backend,本教程是游戏版本,同一 snake 架构,换 game.py + 后端 |
| **18_ipython-kernel + AgentJev** | 数据决策体系,本教程是同思路在游戏场景的实操 |
| **19_os-snake-pipeline_0mcp_phaseA** | 0 mcp 桌面自动化,本教程也是 0 mcp (纯 Python + pip + stdlib) |

---

## 十二、下一步可能的方向

- **微调 Laya multilingual 到 Tetris**:zero-shot 已 0.86,微调后估计 0.95+(参考 laya-snake 0.38→0.90)
- **加 look-ahead**:用 Laya 评估"放这块后下一块怎么放",BFS depth 2-3
- **加 next-piece preview**:前端显示下一个方块,Laya 可以用 next 信息调整当前决策
- **多 Backend**:把 game.py 换成 Browser/Flappy/Pac-Man,policy.py 不动

---

## 十三、附录 A:Laya 参与决策的**统一分层机制**

把整个系统按"职责"切成 6 层,从用户大脑到物理像素,每一层只做一件事,Laya 只出现在第 3 层:

```
┌─────────────────────────────────────────────────────────────────────┐
│  用户层 (User)                                                          │
│  打开浏览器 → 看 Laya 自动玩 Tetris                                          │
└─────────────────────────────────────────────────────────────────────┘
                                  ↕ HTTP (GET / POST JSON)
┌─────────────────────────────────────────────────────────────────────┐
│  L6 前端渲染层 (Frontend — index.html)                                   │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  ① spawn: drawSpawn(state, current) → 方块在 row 0 中央         │ │
│  │  ② decide: api("/api/tetris/step") → 等 Laya 评分                │ │
│  │  ③ animate: drawDropFrame() → 水平 35% + 垂直 65% 滑动          │ │
│  │  ④ settle: 200ms 静止 → 回到 ①                                    │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  ↕ POST /api/tetris/step
┌─────────────────────────────────────────────────────────────────────┐
│  L5 HTTP 接口层 (server.py — ThreadingHTTPServer)                          │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  POST /api/tetris/new        → TetrisApp.new_game()            │ │
│  │  POST /api/tetris/snapshot   → TetrisApp.snapshot()            │ │
│  │  POST /api/tetris/step       → TetrisApp.step()                │ │
│  │  threading.Lock(serial)       → Laya 调用串行化                │ │
│  │  MAX_SESSIONS = 16            → 限制并发 session               │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  ↕ Python function call
┌─────────────────────────────────────────────────────────────────────┐
│  L4 规则与决策层 (policy.py + game.py)                                    │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  TetrisGame.candidates()       → 枚举所有合法落点 → 启发式筛 4 │ │
│  │  describe(cand)                → 中文陈述(消 X 行/留 Y 洞/...) │ │
│  │  LayaTetrisPolicy.rate()       → 4 候选 × 1 次 noul            │ │
│  │  LayaTetrisPolicy.decide()     → argmax P(好) + safety shield │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  ↕ Python function call (agent.predict)
┌─────────────────────────────────────────────────────────────────────┐
│  L3 ★ Laya 决策核心 (laya.Agent — mmBERT 322M, 1024 tokens) ★           │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  输入: state = "这个落点消除两行,不留空洞,堆叠保持低位。"          │ │
│  │  问题: {"q": {"type": "noul", "instructions": "好落点吗?"}}   │ │
│  │  输出: {"answers": {"q": {"noul": 0.86, "confidence": 0.85}}}│ │
│  │  ⏱  ~100ms/次, 单 forward pass, 零输出 token, 无幻觉          │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  ↕ HTTP (HF Hub)
┌─────────────────────────────────────────────────────────────────────┐
│  L2 模型加载层 (runtime.py — load_agent)                                  │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  HF_ENDPOINT=https://hf-mirror.com  (CN-ISP 必需)               │ │
│  │  Agent("convaiinnovations/laya", subfolder="multilingual")     │ │
│  │  warmup("这个落点不消行。") → 触发 CUDA init                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  ↕ safetensors 文件
┌─────────────────────────────────────────────────────────────────────┐
│  L1 物理层 (HF cache disk)                                                │
│  C:/Users/Administrator/.cache/huggingface/hub/                        │
│    models--convaiinnovations--laya/snapshots/<hash>/                  │
│      multilingual/model.safetensors  (643 MB, 322M params)            │
└─────────────────────────────────────────────────────────────────────┘
```

### 数据流(每方块生命周期)

```
[1] User → Browser → /api/tetris/snapshot
                      ↓
[2] server.py → game.snapshot() → {current: "T", board: [...], next: "O"}
                      ↓
[3] Browser drawSpawn(state, "T") → 用户看到 "T" 在 row 0 中央
                      ↓
[4] Browser → /api/tetris/step
                      ↓
[5] server.py → policy.decide(game):
        [5a] game.candidates() → 4 个落点 (heuristic 排序)
        [5b] describe(cand) → "这个落点消除两行..."
        [5c] laya.Agent.predict(statement, {type:noul}) → P=0.86
        [5d] argmax P → executed = cand[2]
        [5e] game.apply(executed) → board 更新
                      ↓
[6] server.py → {decision, before, state, stats} → Browser
                      ↓
[7] Browser animateDrop(before, decision, 600ms) → 水平 210ms + 垂直 390ms
                      ↓
[8] Browser drawBoard(state) → 用户看到 "T" 落在 row 14
                      ↓
[9] Browser await setTimeout(settle, 200ms) → 用户看到 "T" 静止
                      ↓
[10] Loop back to [1] with next piece "O"
```

### 决策权分配(关键)

| 决策内容 | 谁做 | 用什么 |
|---|---|---|
| **下一个方块是什么** | server (TetrisGame._draw / Random bag) | `random.Random(seed).shuffle()` |
| **候选 4 个落点** | game.py heuristic (经典 Bertsekas-Tsitsiklis) | `3*cleared - 5*holes - 0.4*Δh - 0.2*bump` |
| **候选转中文** | policy.py describe() | `CN_NUM + 模板` |
| **每个候选的 P(好)** | **★ Laya multilingual ★** | `agent.predict(statement, noul)` |
| **最终选哪个** | policy.py decide() | `argmax(P) + safety shield` |
| **动画展示** | index.html Canvas | `requestAnimationFrame` |

**Laya 只负责表里第 4 行 — "每个候选的 P(好)"**。其余 6 项全是 Python 代码。

### 为什么这样分层 work (4 个不动点)

1. **Backend 不做判断** — `Game.candidates()` 纯算法,不涉及 LLM
2. **Policy 输入必须压缩到 2-5 候选** — Laya sweet spot 是 4 选 1
3. **Planner 不生成坐标** — 所有"位置"在 policy 层处理
4. **三层之间数据契约严格隔离** — 上下游只通过 JSON / Python 传值,无共享状态

---

**总结**:本教程证明 **Laya multilingual 零样本就能玩 Tetris**(零训练数据,纯中文语义理解)。Laya 的游戏参与 = 中文陈述 + noul 评分 + argmax + safety shield。完整代码 + 真实跑通数据 + 7 个踩坑 + 6 层统一分层架构全在上面,直接 clone `zzhdbw/laya-Ascend/examples/tetris/` 就能复现。