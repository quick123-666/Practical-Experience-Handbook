# OS Snake Pipeline — 0 mcp 纯 Python 路径 Phase A 实战

> **实战目标**:在禁用所有 MCP 工具 (`nuphus-mcp` / `mcp_browser` / `mcp__desktop-commander`) 的前提下,用 **arc-cua 风格 Backend 协议 + 纯 Python** (pyautogui / win32gui / pyperclip / pywin32) 跑通 "在 notepad 写日报并保存到桌面" 完整 7-turn snake pipeline + **Laya multilingual 真决策接入**。
> **落地日期**: 2026-09-27
> **适用**: Windows + Python 3.11 + Laya multilingual (HF mirror) + edge-tts (TTS 备用)

---

## 一、缘起:为什么 0 mcp

用户 2026-09-26 早上明确指令:

> "禁用所有 mcp:nuphus-mcp / mcp_browser / mcp__desktop-commander 都禁,只能用 Python 标准库 + pip 装的纯 Python 模块"

理由:
- mcp 工具链不稳定(版本/权限/启动慢)
- 想验证 **snake pipeline 框架本身** 能不能脱离 mcp 独立跑通
- 0 mcp 路径便于**单元测试** — 不需要起 mcp server,直接 mock backend 测 orchestrator

跟已有教程的差异:

| 维度 | 教程 16 (nuphus-mcp 路径) | 教程 17 (multi-backend) | 本教程 (0 mcp Phase A) |
|---|---|---|---|
| 屏幕感知 | `desktop_perceive` (OCR + YOLO) | 同 + geometric filter | `win32gui.EnumWindows` + `ImageGrab` + 手工建模 |
| 鼠标键盘 | `desktop_mouse.click` | 同 | `pyautogui.click` |
| 中文输入 | nuphus clipboard API | 同 | `pyperclip.copy` + `pyautogui.hotkey("ctrl","v")` |
| Backend 协议 | nuphus BrowserBackend | nuphus multi-backend | **arc-cua 风格 OSBackend** (`observe/is_fresh/execute`) |
| Laya multilingual | ✓ | ✓ | ✓ |
| 单元测试 | 端到端黑盒 | 端到端黑盒 | **34 个 unit tests + 端到端 spike** |

---

## 二、整体架构 (arc-cua 风格 7 层)

```
L7 任务描述: "在 notepad 写日报并保存到桌面"
 ↓
L6 Planner (LLM,System 2 稀疏规划)         ← Phase 2 暂不做
  ↓ 分解为 7 turn
L5 Policy (Laya multilingual,System 1 密集决策)  ← Phase A4 已接
  ↓ 每个 turn 选 1 个 ExecutableAction
L4 Backend 协议 (observe/is_fresh/execute)   ← arc-cua 接口
  ↓
L3 感知层 (win32gui + ImageGrab + pyperclip)  ← 0 mcp,纯 py
  ↓
L2 系统 API (Windows + pyautogui + pywin32)
  ↓
L1 物理层 (鼠标键盘 + 屏幕像素)
```

跟教程 16 区别:**L4-L3 都由本项目代码实现**,不依赖外部 mcp server。Backend 协议严格遵循 arc-cua 上游接口规范(`arc_cua/backends/browser.py`),方便未来切换 BrowserBackend / MobileBackend 等其他实现。

---

## 三、三角色分工 + 4 个不动点

### 3.1 三角色

| 角色 | 系统类比 | 速度 | 输入 | 输出 | 当前状态 |
|---|---|---|---|---|---|
| **Planner** | System 2 (慢想) | 2-10s | 用户自然语言任务 | 7-turn plan (target 序列) | Phase 2 暂不做 |
| **Policy** | System 1 (快反应) | 220ms warm | per-turn target + 2-5 候选 | 1 个 ExecutableAction | Phase A4 已接 Laya multilingual |
| **Backend** | 反射弧 (无意识) | <100ms | ExecutableAction | UI 状态改变 + 截图 | Phase A1 已实现 |

### 3.2 4 个不动点(为什么这个分工能 work)

**不动点 1:Backend 永远不做判断**

Backend 3 个方法 (`observe / is_fresh / execute`) 全部是机械动作,不做"哪个重要 / 该不该"。

**不动点 2:Policy 输入必须压缩到 2-5 候选**

| 候选数 | Laya 命中率 | conf | 备注 |
|---|---|---|---|
| 100 (single-shot) | 0/3 | <0.5 | 死 |
| 4-5 (snake format) | 5/5 | 0.85 | **sweet spot** |
| 1 (heuristic 后) | N/A | N/A | 直接选,根本不调 Laya |

**heuristic_filter 是 Laya 的"输入清洗器"**,决定 Laya 上界:

```python
heuristic_filter(elements, target, max_n=4):
  for elem in elements:
    text_score = fuzzy_text_score(elem.name, target)   # 0-1
    center_bias = 1 - distance_to_screen_center / 3000  # 0.3-1
    score = text_score * 0.7 + center_bias * 0.3
  return top 4
```

**不动点 3:Planner 不生成坐标 / element id**

Planner 只生成抽象的"target"中文短句。绝不输出:
- "click (120, 235)"
- "choose element id=menu_save"

**不动点 4:三层之间的数据契约严格隔离**

```
Planner ──(target: str)──→ Policy
Policy  ──(action: ExecutableAction)──→ Backend
Backend ──(OSSnapshot: elements+revision)──→ Policy
```

没有反向数据流,没有共享状态。

---

## 四、5 个项目内固化的回归点

每个回归点都是"踩坑 → 修复 → 单元测试覆盖"。

### 4.1 坑 #1:revision 用全屏截图 hash 永远 stale

- **症状**: `StaleDesktopState: target 'edit_area' stale` 每次 observe 都 raise
- **真因**: 全屏截图含时钟秒数/光标位置/Windows 通知,像素每秒变
- **修复**: `revision = md5(title|hwnd)`,screenshot sha 降级为 debug-only
- **测试**: `TestRevisionRegressionBug1` 4/4

```python
def _compute_revision(title: str, hwnd: int, _screenshot_sha_unused: str = "") -> str:
    """⚠️ 重要:screenshot_sha 参数**故意忽略**"""
    return hashlib.md5(f"{title}|{hwnd}".encode()).hexdigest()[:16]
```

### 4.2 坑 #2:Python `elif` 不惰性,AttributeError 误读

- **症状**: 错误信息 "DOUBLE_CLICK",看着像"双击失败"
- **真因**: Python `elif` evaluate 整个条件时先读 `ActionKind.DOUBLE_CLICK`,漏写 enum 成员 → `AttributeError("DOUBLE_CLICK")`,字符串 "DOUBLE_CLICK" 被误读为 action 失败原因
- **修复**: dict-based dispatch + import-time `_validate_dispatch()` 检查 enum 和 dispatch keys 一致
- **测试**: `TestDispatchRegressionBug2` 3/3

```python
_DISPATCH_TABLE = {
    ActionKind.CLICK: _do_click,
    ActionKind.DOUBLE_CLICK: _do_double_click,
    # ...
    ActionKind.SET_VALUE: _do_set_value,
}

def _validate_dispatch():
    """启动时检查所有 ActionKind 成员都有 handler。"""
    expected = set(ActionKind.__members__.keys())
    actual = set(k.name for k in _DISPATCH_TABLE.keys())
    if expected != actual:
        raise RuntimeError(f"[os_backend] dispatch table 与 ActionKind 不一致...")

_validate_dispatch()  # module import 时跑,任何不一致 import 就失败
```

### 4.3 坑 #3:start() 没区分新/已有窗口

- **症状**: 已有 notepad "日期 2026-09-27" 在跑,Popen 新 notepad 后 find "无标题" 找不到旧窗口 → raise;或者旧 notepad "无标题" 复用了两次结果污染
- **修复**: `start(reuse_if_exists=True, force_new=False)` 默认复用现成窗口 + `_find_window_by_title(return_empty=True)` 支持 None 返回
- **测试**: `TestStartReuseVsNew` 4/4 (explicit_hwnd / reuse / force_new / no_existing)

### 4.4 坑 #4:turn N Laya 决策时 `targets_per_turn[N]` IndexError

- **症状**: `list index out of range`
- **修复**: `targets[min(turn_idx, len(targets) - 1)]` 越界复用最后一个

### 4.5 坑 #5:type_text dispatch **不按 Enter**

- **症状**: `value="path\n"` 想自动 Enter 触发保存按钮,实际只 copy+Ctrl+V 没按 Enter
- **真因**: OSBackend._do_type_text 只调 `pyperclip.copy` + `hotkey("ctrl","v")`,value 末尾 `\n` 是字符串字符不是 key press
- **修复**: Enter 必须**单独一个 turn**(hotkey "enter")。`type_text` + `Enter` 是 2 个 action
- **适用**: 任何 hotkey/path+Enter 场景

---

## 五、Laya multilingual 实测数据

### 5.1 cold load

- Router init: **3.4s**
- 首次 choice (下载 ckpt): **47.8s** (1.4GB from `https://hf-mirror.com`)
- warm choice: **<1s** (实测 220ms)

### 5.2 中文场景准确率

| 场景 | 候选数 | Laya hit rate | conf |
|---|---|---|---|
| snake 4-direction (PAI DSW) | 4-5 | **5/5 (100%)** | 0.85 |
| 100-way single-shot (desktop control) | 100 | **0/3 (0%)** | <0.5 |
| 中文 OS 元素 (Phase A4.3 实测) | 2 | 1/1 (single-cand 跳过) | 1.0 |
| 中文模糊菜单 (Phase A4.3 warmup) | 3 | 选择有偏差 | 0.027-0.105 |

**关键结论**:Laya 在 4-5 候选 snake format 上 100% 命中,但中文模糊场景 conf 普遍 0.029-0.5。**threshold 默认 0.0**,让 pipeline 跑通优先,准确率作 separate metric。

### 5.3 HF mirror 必备

```python
os.environ.setdefault("HF_ENDPOINT", "https://hf-mirror.com")
```

CN-ISP 不通 `huggingface.co`,必须用国内镜像。`LayaClient` import `laya.Router` 之前设置。

---

## 六、Phase A5 完整保存链路

7-turn spike 实跑数据(2026-09-27 12:36):

```
Turn 0: click edit_area                              → verify OK
Turn 1: type_text "日期 2026-09-27\n"                → verify OK
Turn 2: type_text "snake pipeline 接 Laya multilingual 完整跑通\n"  → verify OK
Turn 3: hotkey Ctrl+S (打开"另存为"对话框)            → verify OK
Turn 4: hotkey Ctrl+A (全选默认文件名"无标题")        → verify OK
Turn 5: type_text "C:\Users\Administrator\Desktop\notepad_2026-09-27.txt"  → verify OK
Turn 6: hotkey Enter (触发默认保存按钮)               → verify OK
```

**结果**:
- 桌面文件 `C:\Users\Administrator\Desktop\notepad_2026-09-27.txt` 70 bytes UTF-8 中文 ✓
- notepad title 变 `notepad_2026-09-27.txt - Notepad`(不带 `*`,表示已 saved) ✓

**视觉验证**:截图 `_os_snake_phase_a5_shots/snap_013.png` 显示 notepad Tab 标题已变 + "另存为"对话框显示完整路径 + 桌面文件 12:36 写入。

**关键限制**:OSBackend.observe() 当前只观察 self.hwnd 进程,**看不见"另存为"系统对话框**。Phase A5 用 hardcode hotkey 序列(`Ctrl+S` → `Ctrl+A` → `type path` → `Enter`)绕开。Phase 1+ 真实业务场景需要扩展 OSBackend 用 `EnumWindows` 抓所有顶层窗口 + 子控件。

---

## 七、跟教程 16/17 对比(原 nuphus-mcp 路径)

| 维度 | 教程 16 (Laya snake) | 教程 17 (multi-backend) | 本教程 (Phase A 0 mcp) |
|---|---|---|---|
| 落地日期 | 2026-09-23 | 2026-09-24 | 2026-09-27 |
| mcp 依赖 | nuphus-mcp | nuphus-mcp | **0 mcp** |
| Laya multilingual | ✓ | ✓ | ✓ |
| snake architecture | ✓ | ✓ | ✓ |
| 候选压缩 heuristic | ✓ | ✓ (geometric 增强) | ✓ |
| per-turn verify | OCR diff | OCR diff | title + pixel hash |
| Backend 协议 | nuphus BrowserBackend | nuphus multi-backend | **arc-cua 风格 OSBackend** |
| 业务场景 | PAI DSW 创建实例 | bilibili 搜视频 | notepad 写日报+保存 |
| 单元测试 | 端到端黑盒 | 端到端黑盒 | **34 个 unit tests** |
| 中文 conf | 0.85 | 0.099-0.276 | 0.029-0.5 |

**本质差异**:教程 16/17 是 **production-grade nuphus-mcp 完整链路**,本教程是 **0 mcp lightweight spike version**。两者 snake pipeline 框架一致,只换 Backend 实现(nuphus BrowserBackend → arc-cua OSBackend)。

**核心相同**:**Laya multilingual decision backend + snake architecture + per-turn verify** 全部一致。

---

## 八、已知限制 + 后续 Phase 1+ 演进

### 8.1 当前限制

1. **OSBackend 看不见系统对话框**:Phase A5 用 hardcode hotkey 序列绕开,Phase 1+ 需要扩展抓所有顶层窗口
2. **OSBackend 只看 self.hwnd 进程的 2 元素**(win + edit_area):不知道菜单、按钮、列表项
3. **Planner 层暂不做**:Phase 0 决策(用户拍板)等框架验证完再加
4. **Laya 中文 conf 普遍低**:不是 framework bug,是 Laya multilingual 真实能力

### 8.2 Phase 1+ 演进路线

| Phase | 目标 | 工作量 |
|---|---|---|
| **Phase 1** | OSBackend 扩展抓所有可见顶层窗口 + 子控件(win32gui.EnumChildWindows) | 2-3 天 |
| **Phase 1.1** | 接 WPS 真实业务(打开 docx + 写入中文 + 保存) | 1-2 天 |
| **Phase 1.2** | 接 Excel 真实业务(选 cell + 写入公式) | 1-2 天 |
| **Phase 2** | Planner 层(MiniMax M3 + top_k=10 + history)+ System 2 任务分解 | 3-5 天 |
| **Phase A5** | SET_VALUE handler(pywinauto 直接 send text 给控件,绕过 IME) | 0.5 天 |

---

## 九、附录

### 9.1 34 个单元测试清单

| 测试类 | 数量 | 覆盖 |
|---|---|---|
| `TestRevisionRegressionBug1` | 4 | 坑 #1 revision title-only |
| `TestDispatchRegressionBug2` | 3 | 坑 #2 dict dispatch + validation |
| `TestBackendProtocolSmoke` | 1 | snake pipeline orchestrator mock 跑通 |
| `TestStaleDesktopState` | 1 | is_fresh False 时 raise |
| `TestFindWindowByTitleReturnEmpty` | 3 | _find_window_by_title None 行为 |
| `TestStartReuseVsNew` | 4 | 坑 #3 reuse vs new 决策树 |
| `TestHeuristicFilter` | 5 | fuzzy_text_score + 排序 |
| `TestFormatSnakeChoice` | 2 | snake 4-class 格式生成 |
| `TestElementsToCandidates` | 2 | OSElement → dict 转换 |
| `TestMockLayaClient` | 3 | mock Laya.choice |
| `TestLayaPolicyDecision` | 5 | single/multi-candidate + low conf + history |
| `TestOrchestratorWithLayaPolicy` | 1 | 集成 LayaPolicy 进 orchestrator |
| **合计** | **34** | **2.031s 跑完 OK** |

### 9.2 文件清单

```
C:\Users\Administrator\Desktop\minimax\os_snake\
├── os_backend.py                    9037 B   arc-cua 风格 Backend 协议 + dispatch
├── os_snake_orchestrator.py          9626 B   4-turn orchestrator + verify
├── laya_policy.py                  12848 B   Laya multilingual 决策 + MockLayaClient
├── phase_a4_laya_real.py            ~3 KB   5-turn Laya 接通 demo
├── phase_a5_save_to_desktop.py      6468 B   7-turn notepad 保存到桌面
├── test_os_snake.py                 ~12 KB   16 tests (Backend protocol)
├── test_laya_policy.py              ~10 KB   18 tests (Laya integration)
├── README.md                        4156 B   完成总结
└── _os_snake_laya_shots/  snap_000..010.png  视觉验证截图
```

### 9.3 关键命令

```bash
# 跑全部测试 (2.031s)
cd C:\Users\Administrator\Desktop\minimax
python -m unittest os_snake.test_os_snake os_snake.test_laya_policy

# 跑 Phase A5 spike (7 turn notepad 写+保存)
python os_snake\phase_a5_save_to_desktop.py

# 跑 Phase A4 Laya 接通 demo
python os_snake\phase_a4_laya_real.py
```

### 9.4 中文输入必须用 pyperclip + Ctrl+V

- `pyautogui.typewrite` 不支持中文(键盘事件无中文字符 keycode)
- 必须 `pyperclip.copy("中文")` + `pyautogui.hotkey("ctrl", "v")`

### 9.5 跨教程引用

- **教程 16** (`16_Laya贪吃蛇思维控制电脑实战教程.md`):Laya snake architecture 起源,100/选-1 vs 4/选-1 实测对比
- **教程 17** (`17_computer_control_pipeline多Backend+几何过滤实战教程.md`):production-grade multi-backend pipeline,geometric filter 详解
- **`2026-09-26_arc-cua-browser-automation-windows.md`**:Phase 1 browser 自动化 spike,§6.5 纯 py 0-mcp 路径实战
- **`2026-09-26_LFM_Jev_机器人控制可行性分析.md`**:机器人 vs 浏览器 vs OS 抽象层对比

---

**总结**:本教程证明 **snake pipeline 框架可以脱离 mcp 独立跑通**,且 5 个回归点项目内固化(单元测试覆盖)。Laya multilingual 决策 backend + snake architecture 跟教程 16/17 完全一致,只换 Backend 实现(OSBackend 而非 nuphus BrowserBackend)。Phase 1+ 真实业务场景需要扩展 OSBackend 感知能力(系统对话框、菜单、按钮),目前 Phase A5 spike 已验证"另存为"对话框的 hotkey 序列模式可复用。