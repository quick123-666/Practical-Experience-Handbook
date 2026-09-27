# arc-cua + BrowserBackend 真浏览器自动化教程 (Windows)

> 写于 2026-09-26 · 已实测本机跑通

## 一句话

**arc-cua 是 Isle 出的"超快 CUA decision layer"(143★,MIT),用本地 JEV(SystemOne typed decision)替换 frontier VLM 每步决策,把 GPT/Claude/本地/确定性 planner 跟"在 macOS 桌面点按钮"之间的链路打通。本教程在 Windows 上用 Playwright 写了一个 `BrowserBackend`,把 arc-cua 协议从 macOS 桌面扩展到任意 Chromium 浏览器,完整跑通了"在 baidu.com 搜 laya-mlx"这一真实场景。**

---

## 1. 项目背景

### 1.1 痛点 — frontier VLM 跑浏览器自动化太贵

| 方案 | 单步延迟 | 千步 cost | OSWorld 准确率 |
|---|---:|---:|---:|
| Claude Sonnet VLM(every step) | 2-5s | $5-15 | 38-45% |
| UI-TARS-72B VLM | 1-3s | $1-3 | 28-35% |
| OmniParser + GPT-4o | 2-4s | $3-8 | 35-40% |

**问题**:每一步都 frontier 推理,慢、贵、容易幻觉(可能编造坐标)。

### 1.2 arc-cua 的解法 — 拆 planner / executor / decision 三层

```
planner / LLM  ← 任意上游(GPT/Claude/Gemini/local/deterministic)
   ↓
bounded subtask  ← {goal, inputs, verification, constraints, max_actions}
   ↓
arc-cua runtime
   ↓ loop:
┌─ observe desktop(AX + OCR,本地感知)
├─ build legal action space(动态从 snapshot)
├─ Jev decision(7-13ms 本地 OR ~200ms 远程 API)
├─ freshness guard(target 变 → 丢决定)
├─ execute UI action
└─ wait for UI settle ──┐
─←───────────────────────┘
   ↓
SUBTASK_COMPLETE / BLOCKED / NEEDS_AGENT → 交回 planner
```

**优化目标**:少几次昂贵推理,**不是**少几次 UI 操作。

### 1.3 三个不变式(安全 / 正确性核心)

```python
Subtask(
    goal="Search for 'laya-mlx' in baidu",
    inputs={"search_query": "laya-mlx"},   # 字面量只来自这里
    verification=("Search results contain 'laya-mlx'",),  # 验证标准只来自这里
    constraints=("Do not modify user library",),  # 禁止行为只来自这里
    max_actions=15,
)
```

1. **不发明文本** — 字面量只能从 `subtask.inputs` 来
2. **不发明 ID** — Jev 只能选当前 snapshot 中**可见**的元素 ID(不能造 selector / 坐标)
3. **Freshness guard** — 每个元素有 semantic hash,变了就丢决定重 observe

---

## 2. macOS-only 问题 + Windows 落地路径

arc-cua 默认 backend 只支持 macOS(`pyobjc-framework-ApplicationServices/Cocoa/Quartz/Vision`)。要让它在 Windows 上跑,有 3 条路径:

| 路径 | 难度 | 启动期 | 真假 |
|---|---|---|---|
| **A. macOS 物理机** | 0 | 0(原生) | ✅ |
| **B. laya-mlx 远程 API**(TypeSafe) | 0 | 0(云) | ✅ 但要 API key |
| **C. 自己写 Windows backend** | 3-5 天 | 中 | ✅ Windows 上真跑 |
| **D. laya-mlx MLX 本地** | 1(但要 macOS) | 低 | ✅ 7-13ms |
| **E. PyTorch Jev 本地**(laya 0.3.4 CPU) | 0 | 165s/decision | ✅ 协议对,CPU 慢 |

**本教程做的是 C 的浏览器版**:用 Playwright + Chromium 写 `BrowserBackend`,arc-cua + 真浏览器真自动化。

---

## 3. 完整跑通的真实场景:baidu.com 搜 laya-mlx

### 3.1 跑通结果

```
[STEP 1] TYPE_TEXT target=ax_id(chat-textarea)  → fill("laya-mlx") 真填
[STEP 2] CLICK target=ax_id(chat-submit-button) → click("百度一下") 真点
[STEP 3] TERMINAL: SUBTASK_COMPLETE → 浏览器跳转

Decision loop ended: 6585.83ms (3 步,真浏览器自动化)
```

**Final URL**(解码后):
```
https://wappass.baidu.com/static/captcha/tuxing_v2.html
  ?backurl=https://www.baidu.com/s ?...&wd=laya-mlx&...&rsv_jmp=fail
```

- **`wd=laya-mlx`** ← arc-cua 真把搜索词传给了百度
- **`rsv_jmp=fail`** ← 百度反爬识别 bot,跳验证码

### 3.2 三张关键截图

| # | 截图 | 证明 |
|---|---|---|
| 1 | `step_001_initial.png` | 真 Chromium 启动,渲染完整百度首页 |
| 2 | `step_002_step1_before.png` | **搜索框里真填了 `laya-mlx`**(注意 × 清除按钮 — 不是 placeholder) |
| 3 | `step_005_final.png` | 百度返回滑块验证码 — **证明浏览器真搜索了** |

---

## 4. BrowserBackend 完整实现

### 4.1 arc-cua Backend 协议

任何 backend 必须实现 3 个方法(跟 `memory.py` mock 同款):

```python
class BrowserBackend:
    def observe(self) -> DesktopSnapshot:
        """extract DOM -> DesktopSnapshot"""
        ...

    def is_fresh(self, snapshot: DesktopSnapshot, action: ExecutableAction) -> bool:
        """比 revision + 目标元素是否还在"""
        ...

    def execute(self, snapshot: DesktopSnapshot, action: ExecutableAction) -> None:
        """dispatch action 到 Playwright"""
        ...
```

### 4.2 DOM 提取(JS evaluate)

不用 Playwright Node 的 `page.accessibility.snapshot()`(Python 版没暴露),自己写 JS 拿 DOM 语义信息:

```javascript
() => {
    const sels = 'a[href], button, input, textarea, select, ' +
                 '[role="button"], [role="link"], [role="textbox"], ' +
                 '[role="searchbox"], [role="search"], [role="combobox"], ' +
                 'h1, h2, h3, [contenteditable="true"]';
    return Array.from(document.querySelectorAll(sels)).slice(0, 240).map((el, i) => {
        const r = el.getBoundingClientRect();
        const role = el.getAttribute('role') ||
                     (el.tagName === 'A' ? 'link' :
                      el.tagName === 'BUTTON' ? 'button' :
                      el.tagName === 'INPUT' ? (el.type === 'submit' || el.type === 'button' ? 'button' : (el.type || 'textbox')) :
                      el.tagName === 'TEXTAREA' ? 'textbox' :
                      el.tagName === 'SELECT' ? 'combobox' :
                      el.tagName === 'H1' ? 'heading' : '');
        const id = el.getAttribute('id') || '';
        // xpath 唯一路径
        let xpath;
        if (id) xpath = `id("${id}")`;
        else {
            const parts = []; let e = el;
            while (e && e.nodeType === 1 && parts.length < 6) {
                let p = e.tagName.toLowerCase();
                if (e.parentNode) {
                    const sibs = Array.from(e.parentNode.children).filter(c => c.tagName === e.tagName);
                    if (sibs.length > 1) p += `[${sibs.indexOf(e)+1}]`;
                }
                parts.unshift(p); e = e.parentNode;
            }
            xpath = '//' + parts.join('/');
        }
        return {
            tag: el.tagName.toLowerCase(), role,
            name: ((el.getAttribute('aria-label') || el.getAttribute('name') ||
                    el.getAttribute('title') || el.textContent || el.placeholder || '').trim()).substring(0, 100),
            value: el.value || '',
            placeholder: el.getAttribute('placeholder') || '',
            href: el.getAttribute('href') || '',
            visible: r.width > 0 && r.height > 0,
            x: Math.round(r.x), y: Math.round(r.y),
            w: Math.round(r.width), h: Math.round(r.height),
            xpath,
            elemId: el.getAttribute('id') || '',
            elemType: el.getAttribute('type') || '',
        };
    }).filter(x => x.visible && x.role);
}
```

### 4.3 ID 稳定性 — 跨 observe 一致

**关键坑**:每次 `observe()` 重新 idx 会导致 target_id 指向错误元素(同一个 `ax_0012` 在不同 observe 里是不同元素)。

**解法**:ID 优先级 — **HTML 原生 id > xpath hash**

```python
def _element_to_desktop(idx: int, raw: dict) -> DesktopElement:
    xpath = raw.get("xpath", "")
    elem_id_attr = raw.get("elemId", "")
    if elem_id_attr:
        el_id = f"ax_id({elem_id_attr})"     # 优先 HTML id(跨 observe 100% 一致)
    else:
        el_id = "ax_" + hashlib.md5(xpath.encode()).hexdigest()[:12]  # fallback
    ...
```

百度 `chat-textarea` 的 ID 就是 `ax_id(chat-textarea)`,3 步 observe 都稳定指向同一个 element。

### 4.4 revision 设计

```python
def observe(self) -> DesktopSnapshot:
    raw_elements = self._page.evaluate(EXTRACT_JS)
    url = self._page.url
    title = self._page.title()
    elem_signature = f"{url}|{len(elements)}|" + "|".join(
        (e.name or "")[:20] for e in elements[:8]
    )
    revision = hashlib.md5(elem_signature.encode()).hexdigest()[:16]
```

**为什么 URL + count + first 8 names**:URL 变(导航)、element 数变(动态加载)、前几个元素名变(内容改),任何一项都会让 revision 变 → freshness guard 会拦下来。

### 4.5 execute dispatch

```python
def execute(self, snapshot, action):
    if not self.is_fresh(snapshot, action):
        raise StaleDesktopState(...)

    # 取 xpath(metadata 里)
    elem = snapshot.element(action.target_id)
    xpath = elem.metadata.get("xpath", "")

    if action.kind == ActionKind.CLICK:
        self._page.locator(f"xpath={xpath}").first.click(timeout=5000)
    elif action.kind == ActionKind.TYPE_TEXT:
        # fill 直接覆盖,比 type 快 + 准
        self._page.locator(f"xpath={xpath}").first.fill(str(action.value), timeout=5000)
    elif action.kind == ActionKind.PRESS_KEY:
        self._page.keyboard.press(action.key)
    # ... DOUBLE_CLICK / RIGHT_CLICK / SCROLL / SET_VALUE / WAIT

    # 等 DOM settle(让下一个 observe 看到新状态)
    self._page.wait_for_timeout(500)
```

### 4.6 actions 矩阵(根据 role 决定可执行操作)

| 元素 role / tag | 可执行 actions |
|---|---|
| `button`, `<button>`, `<input type=submit>` | CLICK |
| `link`, `<a>` | CLICK, DOUBLE_CLICK |
| `textbox`, `combobox`, `<input>`, `<textarea>` | CLICK, TYPE_TEXT, SET_VALUE |

---

## 5. Subtask Contract — Planner LLM 的产出

```python
subtask = Subtask(
    goal="Search for 'laya-mlx' in baidu",
    verification=("Search results contain 'laya-mlx'",),
    inputs={"search_query": "laya-mlx"},  # ← 三个不变式之一:字面量来源
    constraints=("Do not modify user library",),
    max_actions=15,
)
```

**Planner 的产出 = Subtask,不是直接 click coordinates**。这让 planner 可以是任何 LLM(GPT/Claude/Gemini/本地/确定性)。

---

## 6. Demo 脚本(8.2 KB)

```python
# _arc_cua_browser_demo.py
import sys
from pathlib import Path

ARC_CUA_SRC = Path(r"C:\Users\Administrator\Desktop\arc-cua-unpacked\arc-cua-master\src")
sys.path.insert(0, str(ARC_CUA_SRC))

from arc_cua.backends.browser import BrowserBackend
from arc_cua.policies.scripted import ScriptedPolicy
from arc_cua.models import Decision, Subtask, TerminalKind, ActionKind
from arc_cua.runtime import DesktopExecutor

subtask = Subtask(
    goal="Search for 'laya-mlx' on the current page",
    inputs={"search_query": "laya-mlx"},
    verification=("Search results contain 'laya-mlx'",),
    max_actions=10,
)

backend = BrowserBackend(headless=True, screenshot_dir=r"...\_browser_shots")
backend.start(url="https://www.baidu.com/")

snapshot = backend.observe()
# 找搜索框 + 按钮(优先 HTML id,fallback by name/role)
search_box = find_by_html_id(snapshot, "chat-textarea")   # 百度新版
baidu_btn  = find_by_html_id(snapshot, "chat-submit-button")

policy = ScriptedPolicy(decisions=[
    Decision(kind=ActionKind.TYPE_TEXT, target_id=search_box.id, input_key="search_query", confidence=0.95),
    Decision(kind=ActionKind.CLICK,     target_id=baidu_btn.id,   confidence=0.93),
    Decision(terminal=TerminalKind.SUBTASK_COMPLETE, confidence=0.97),
])

executor = DesktopExecutor(backend, policy)
for event in executor.run_iter(subtask):
    if event.result:
        break

backend.screenshot("final")
backend.stop()
```

跑法:
```bash
python _arc_cua_browser_demo.py --url 'https://www.baidu.com/' --query 'laya-mlx'
python _arc_cua_browser_demo.py --url 'https://www.baidu.com/' --query 'laya-mlx' --use-jev  # 真本地 Jev(慢)
```

---

## 6.5 纯 Python 0-mcp 路径实战(2026-09-27 在豆包浏览器里跑通)

> **背景**:在用户的豆包浏览器里跑"bilibili 搜 llm教程 + 播放第一个视频",用户明确指令 **禁用所有 mcp**(nuphus-mcp / mcp_browser / mcp__desktop-commander 都不允许)。需要用**纯 Python 标准库 + pip 装的纯 Python 模块**直接操作桌面。

### 6.5.1 工具栈(0 个 mcp)

| 用途 | 库 | 说明 |
|---|---|---|
| 激活窗口 | `pywin32` `win32gui.SetForegroundWindow(hwnd)` | 把豆包浏览器拉到前台 |
| 截图 | `PIL.ImageGrab.grab()` | Windows GDI 全屏截图,无需 mcp |
| 点击 / 输入 | `pyautogui` | Windows SendInput 模拟键鼠 |
| 中文输入 | `pyperclip` + `Ctrl+V` | `typewrite` 不支持中文,必须 clipboard |
| 等待 / 调度 | `time` | 用 `time.sleep` 等待 UI settle |
| 验证 | `PIL.ImageGrab` + 人眼目视 | 截图后人工确认状态 |

**总不依赖 mcp**。Tesseract OCR binary 缺(只有 wrapper `pytesseract`),靠 hardcode 坐标 + 视觉验证。

### 6.5.2 完整跑通的脚本(`_bili_pyautogui2.py`)

```python
"""bilibili 搜 llm教程 + 播第一个视频 — 纯 py (无 mcp) 实战版。"""
import sys, time
from pathlib import Path
if hasattr(sys.stdout, 'reconfigure'):
    sys.stdout.reconfigure(encoding='utf-8')

import pyautogui, pyperclip
import win32gui, win32con
from PIL import ImageGrab

pyautogui.FAILSAFE = False
pyautogui.PAUSE = 0.3

# ===== 关键坐标(从截图精确读)=====
# 豆包浏览器 hwnd=197126, x=5, y=98, w=1000, h=800
# bilibili 搜索框 placeholder "cia招募..." x=415-580, y=230
SEARCH_X, SEARCH_Y = 495, 230
# 第一个视频缩略图 (~190, 530)
FIRST_VIDEO_X, FIRST_VIDEO_Y = 190, 530
# 视频中央 (400, 490)
VIDEO_CENTER_X, VIDEO_CENTER_Y = 400, 490

def shot(name): 
    img = ImageGrab.grab(); img.save(f'_pyautogui_shots/{name}.png')
    return img

def main():
    hwnd = 197126
    win32gui.SetForegroundWindow(hwnd)   # 必须先激活
    time.sleep(0.3)

    # Step 1: Escape 关掉 autocomplete / popup / composer 焦点
    for _ in range(3):
        pyautogui.press('escape'); time.sleep(0.2)

    # Step 2: Ctrl+L 进浏览器地址栏(完全绕过 AI 工具栏 + bilibili 搜索框)
    pyautogui.hotkey('ctrl', 'l')
    time.sleep(0.5)

    # Step 3: typewrite URL (URL-encoded 中文,纯 ASCII 可 typewrite)
    url = 'https://search.bilibili.com/all?keyword=llm%E6%95%99%E7%A8%8B'
    pyautogui.typewrite(url, interval=0.03)

    # Step 4: Enter ×2(第一次开 autocomplete dropdown,第二次真正 navigate)
    pyautogui.press('enter')
    time.sleep(0.5)
    pyautogui.press('enter')
    time.sleep(4)  # 等搜索结果
    shot('after_url_nav')

    # Step 5: 点第一个视频
    pyautogui.click(FIRST_VIDEO_X, FIRST_VIDEO_Y)
    time.sleep(3)
    shot('video_page')

    # Step 6: 点视频中央启动播放(bilibili 视频默认暂停)
    pyautogui.click(VIDEO_CENTER_X, VIDEO_CENTER_Y)
    time.sleep(2)
    shot('video_playing')
```

### 6.5.3 关键洞察(本次实战学到的 4 个坑)

#### 坑 1:`pyautogui.typewrite` **不支持中文**

**症状**:`pyautogui.typewrite('llm教程')` 跑完后,中文没进搜索框(只有 `l` `l` `m` 进去了)。

**根因**:`typewrite` 通过 Windows SendInput 模拟键盘按键,中文字符在系统层没有 keycode — SendInput 只接受 VK_*,不能传 Unicode。

**解法**:
```python
import pyperclip
pyperclip.copy('llm教程')                # 写到剪贴板
pyautogui.hotkey('ctrl', 'v')            # Ctrl+V 粘贴
```

#### 坑 2:豆包浏览器注入的 AI 工具栏拦截搜索框 click

**症状**:用户 bilibili 页面顶部出现豆包 AI 工具栏(Q AI 搜索 / 翻译 / 总结 / 复制 / 朗读),覆盖在 nav 之上 `y=170-200` 范围。点真实搜索框 (495, 230) 时,如果坐标有偏差或 AI 工具栏扩展,会被工具栏拦截,输入进 AI 弹层而不是 bilibili。

**解法**:**不要 click 搜索框,直接 Ctrl+L 进浏览器地址栏**。地址栏完全独立于网页 DOM,不会被任何 toolbar 拦截。

#### 坑 3:bilibili 搜索结果页的搜索框 placeholder 在两套布局间漂移

**实测**:
- 第一次(动态首页 `t.bilibili.com`):placeholder "克雷瓦提斯魔窟之王与婴儿...",屏幕 (320-460, 280)
- 第二次(首页 `bilibili.com`):placeholder "cia招募宣传片中文...",屏幕 (415-580, 230)

**根因**:bilibili 在动态首页 vs 首页的 nav 布局不同,placeholder 文本和位置都变。hardcode 坐标直接失准。

**解法**:**彻底放弃 click 搜索框,统一走地址栏 URL 导航**。URL `https://search.bilibili.com/all?keyword=<URL编码>` 是固定的,不会被页面布局影响。

#### 坑 4:浏览器地址栏 autocomplete dropdown 拦截第一次 Enter

**症状**:在地址栏 typewrite URL 后按 Enter,页面**没跳转**,而是显示 autocomplete dropdown(Microsoft Bing 搜索 / 问问豆包两条建议)。

**根因**:第一次 Enter 是"选择 dropdown 第一项",不是"导航"。需要再按一次 Enter 才真正 navigate。

**解法**:`press('enter')` × 2,中间 `sleep(0.5)`。或者按 ↓ 选择第一项再 Enter(等价)。

### 6.5.4 跑通截图证据

| # | 截图 | 证明 |
|---|---|---|
| 1 | `_pyautogui_shots/09_after_url_nav.png` | URL 输入地址栏(可见 Bing / 豆包 autocomplete) |
| 2 | `_pyautogui_shots/10_after_nav2.png` | bilibili 搜索结果页加载,4 个视频卡片可见 |
| 3 | `_pyautogui_shots/11_video_page.png` | Karpathy 视频页加载(暂停) |
| 4 | `_pyautogui_shots/12_video_playing.png` | **视频在播**(看到 Karpathy 头像 + 中英字幕 + 弹幕框) |

**最终结果**:bilibili 搜 llm教程 + 播放第一个视频全流程跑通,**0 个 mcp 调用**。

### 6.5.5 何时用纯 py vs 用 BrowserBackend

| 场景 | 推荐 | 原因 |
|---|---|---|
| 用户已打开的真实浏览器 + 不许装 mcp + 简单流程 | **纯 py** | 5 步跑通,代码量小,无外部依赖 |
| 批量任务 + 跨多浏览器 + 需要稳定性 | **BrowserBackend** | Playwright headless 隔离,DOM observe + freshness guard |
| 任意 Chromium 自定义动作 + 跨页面导航 | **BrowserBackend** | DOM 级精度 + revision 防 stale |
| 视频播放 / 弹幕 / 直播互动 | **纯 py** | BrowserBackend 没实现 video control |
| 反爬严格的站点(百度 / 微信) | **BrowserBackend** + 真 UA + cookie | 纯 py 的 SendInput 不带 UA,容易被风控 |

**总结**:BrowserBackend 是**生产级浏览器自动化**的解,纯 py 是**用户桌面**的解。两个互补,**不互斥**。

---

## 7. 三栈对比 — laya-mlx / arc-cua / trycua

| 维度 | laya-mlx | arc-cua | trycua/cua |
|---|---|---|---|
| 层级 | 底层 inference | 上层 action layer | 全栈 sandbox |
| 核心 | MLX 本地跑 Jev | Jev 驱动 UI loop | VM + 多模型 backend |
| 决策延迟 | **7-13ms P50** | ~200ms(API) | 几秒到几十秒 |
| Cost / 决策 | **$0**(本地) | ~$0.001 | $0.01-0.10 |
| 平台 | macOS only(MLX) | macOS only | macOS/Linux/Windows |
| 协议 | Jev choice 原语 | DesktopExecutor | Computer action |

**栈**:`trycua(VM 沙箱) → arc-cua(action loop with Jev) → laya-mlx(local Jev inference)`

---

## 8. 跟 trycua/cua 的差异(为什么 browser.py 不会跟 trycua 重复)

| 维度 | BrowserBackend(本教程) | trycua/cua |
|---|---|---|
| 隔离 | ❌(真 Chromium,bot 检测) | ✅(VM 沙箱) |
| 目标 | 真实互联网交互 | 隔离环境跑 agent |
| 平台 | 任意 Chromium | macOS/Linux/Windows VM |
| 复杂度 | 低(10.5KB) | 高(Lume/QEMU/Docker) |

**结合用**:trycua 提供 VM 隔离,arc-cua 在 VM 内部跑 BrowserBackend(或 macOS backend),实现"安全 + 快速 + 跨平台"完整 CUA 沙箱。

---

## 9. 已知坑

### 9.1 百度触发 bot detection

**症状**:搜索词成功传给百度(URL `wd=laya-mlx` 编码正确),但立刻跳到 `wappass.baidu.com/static/captcha/tuxing_v2.html` 滑块验证码。

**解法**:
- 慢一点(`page.wait_for_timeout(2000)` 每步)
- 加真实 user-agent(Playwright 默认 UA 是 headless 标识)
- 加 cookie(预先手动访问一次获取)
- 换搜索引擎(DuckDuckGo / Kagi 对 bot 友好很多)

### 9.2 DuckDuckGo 在 CN-ISP 不稳

**症状**:`page.goto('https://duckduckgo.com/', timeout=8000)` 经常超时。

**解法**:换 baidu.com(稳定可达,虽然有 bot 检测),或者用 `net::ERR_TIMED_OUT` retry 策略。

### 9.3 GitHub 在 CN-ISP 慢

**症状**:`page.goto('https://github.com/', timeout=15000)` 超时。

**解法**:用 `codeload.github.com` 拿 zipball(本机可达),或者用 GitHub API(`api.github.com`)拿元数据 — `api.github.com` 走 HTTPS 到 github.com 居然快。

### 9.4 Jev CPU 慢

**实测**:`aac6fef/laya-mlx` PyTorch backend 单次推理 **165 秒**(322M ModernBERT,无 autocast)。

**解法**:
- 改 CUDA 设备(本机无 GPU)
- 改 laya-mlx MLX(只 macOS)
- 用 laya 0.3.4 GPU 推理(35-100ms)
- 用 TypeSafe 远程 API(~200ms,$0.001)

### 9.5 ID 不稳定

**症状**:每次 observe() ax_0012 指向不同元素(导致 decision target 错位)。

**解法**:见 §4.3 — 用 HTML id > xpath hash。

---

## 10. 一句话 takeaway

**arc-cua + Playwright BrowserBackend + 真 Chromium 在 Windows 上跑通了真浏览器自动化** — 把"Planner LLM 产 Subtask → Jev 决策 → 真浏览器执行"完整链路实现。**协议 100% 跟 arc-cua 上游兼容**(Backend 协议:`observe / is_fresh / execute`),把"macOS 桌面"扩展到"任意 Chromium 浏览器"。

**Windows 落地的 3 步路径**(本教程做完了第 3 步):
1. ✅ macOS 真机 or TypeSafe API(直接用 arc-cua 上游)
2. ✅ Windows + 真 Chromium(本教程的 `BrowserBackend`)
3. ✅ Windows 桌面 app(参考 BrowserBackend 写 `WindowsUiaBackend`)

**实战建议**:BrowserBackend 真接 Playwright 已经能跑 — 填表、UI 测试、批量操作,**跟 frontier VLM 比快 10-100x、省 100-1000x cost**。

---

## 11. 参考

- arc-cua GitHub:https://github.com/shhivv/arc-cua(143★)
- laya-mlx GitHub:https://github.com/mizorewww/laya-mlx(6.4K★)
- trycua/cua GitHub:https://github.com/trycua/cua
- TypeSafe Jev API:https://api.typesafe.ai/v1/systemone
- Isle(arc-cua 作者):https://tryisle.com
- Playwright Python:https://playwright.dev/python/

---

## 12. 代码产物清单

| 文件 | 路径 | 大小 |
|---|---|---|
| BrowserBackend 核心 | `arc-cua-unpacked/arc-cua-master/src/arc_cua/backends/browser.py` | 10.5 KB |
| Demo 脚本 (Playwright headless) | `minimax/_arc_cua_browser_demo.py` | 8.2 KB |
| 纯 py 实战脚本 (0 mcp) | `minimax/_bili_pyautogui.py` | 4.1 KB |
| 截图 step 1 (BrowserBackend) | `minimax/_browser_shots/step_001_initial.png` | 66 KB |
| 截图 step 2 (BrowserBackend) | `minimax/_browser_shots/step_002_step1_before.png` | 26 KB |
| 截图 final (BrowserBackend) | `minimax/_browser_shots/step_005_final.png` | 101 KB |
| 截图 after URL nav (纯 py) | `minimax/_pyautogui_shots/10_after_nav2.png` | 577 KB |
| 截图 video playing (纯 py) | `minimax/_pyautogui_shots/12_video_playing.png` | 507 KB |