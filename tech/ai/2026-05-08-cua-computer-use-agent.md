---
title: CUA — Computer-Use Agent Infrastructure
type: clipping
source: GitHub - trycua/cua
url: https://github.com/trycua/cua
date: 2026-05-08
category: ai-agent
description: trycua/cua 深度技術分析：macOS SkyLight + AX 內部、Windows UIA 實作、Linux Xvfb 替代、三平台對照、agent stack 決策樹
captured: 2026-04-25
updated: 2026-05-08
tags: [computer-use, agent, automation, sandbox, vm, lume, mcp, claude-code, xvfb, linux, windows, playwright, uia]
---

# CUA — Computer-Use Agent Infrastructure

> **來源**：[trycua/cua · GitHub](https://github.com/trycua/cua)（擷取於 2026-04-25）

## 簡介

CUA 是開源的 Computer-Use Agent 基礎設施，提供沙盒（Sandbox）、SDK、driver 與 Benchmark，用於訓練、部署、評估能控制完整桌面環境的 AI Agent。

GitHub Stars：**15,732**｜Forks：**975**｜主要語言：HTML、Python、Swift（cua-driver）

cua-driver 12 天連發 5 版（v0.1.1 → v0.1.5），專案進入快速 iterate 階段。

---

## 核心元件

| 元件 | 說明 |
|------|------|
| **Cua Driver** | 背景自動化，控制原生 macOS 應用，不搶佔游標焦點。Speaks MCP over stdio |
| **Cua Sandbox** | 統一 API，跨 OS 管理 VM 與容器（Python 3.11+） |
| **CuaBot** | 協作式沙盒環境，原生桌面視窗 + H.265 串流、共享 clipboard、audio |
| **Cua-Bench** | Benchmark 套件與強化學習（RL）環境，整合 OSWorld、ScreenSpot、Windows Arena |
| **Lume** | macOS/Linux VM 管理，Apple Virtualization.Framework，Apple Silicon 近原生效能 |

---

## 任務光譜：你真的需要 cua 嗎？

寫 agent 之前先看任務分布。Production agents 80% 場景，**headless Chrome + CDP 就解決**，比走 cua 路線更穩、更平行、更便宜。

| 任務類型 | 解法 | 頻率 |
|---------|------|------|
| 純網頁（form fill / 爬資料 / OAuth） | Playwright / CDP headless | ~70% |
| Web 但偵測 headless（金流、ticketing） | Headed Chrome in Xvfb container | ~10% |
| Web + browser native dialog（file picker / print） | Playwright file chooser API | ~5% |
| 原生桌面 app（Office / Excel / ERP / 自寫 desktop app） | OS-specific（cua / UIA / Linux X11） | ~10% |
| 跨 app 工作流（File Explorer drag → app） | OS-level 混搭 | ~3% |
| 遊戲 / 3D / DirectX / OpenGL | 沒好解，HID stream + frontmost | ~2% |

**→ 70–85% 的 agent 任務 headless Chrome 解決，cua 的價值集中在剩下 15–30% 需要 OS 級桌面控制的場景。**

---

## 平台支援

### Host（agent 跑在哪）

| 元件 | 限制 |
|------|------|
| cua-driver | **macOS only**（用 macOS 系統 API 控制原生 app） |
| Lume | **Apple Silicon only**（M1/M2/M3） |
| cua-sandbox | 跨平台（Linux / Windows / macOS host 都能跑） |

完整 stack 最佳體驗：**Apple Silicon Mac**。Linux/Windows host 跑 sandbox 可，但拿不到 cua-driver 的 macOS 原生 app 控制與 Lume 的近原生 VM 效能。

### Guest（agent 控制誰）

cua-sandbox 統一 API 支援多平台 target：

```python
async with Sandbox.ephemeral(Image.linux()) as sb:    # Linux
async with Sandbox.ephemeral(Image.macos()) as sb:    # macOS
async with Sandbox.ephemeral(Image.windows()) as sb:  # Windows
async with Sandbox.ephemeral(Image.android()) as sb:  # Android
# BYOI: .qcow2 / .iso 自訂映像
```

---

## 整合方式（自寫 agent 接 CUA）

CUA 對自寫 agent 的角色是「Computer Use 工具的後端」 — agent 負責 LLM + planning，CUA 提供桌面/視窗操作。三條路：

### Path A — MCP server（最通用）

cua-driver 本身是 MCP server，agent 走 MCP client 連線，把 cua-driver 提供的 tools 加進 LLM 的 tool list。

```bash
cua-driver mcp
# 或 Claude Code:
claude mcp add --transport stdio cua-driver -- cua-driver mcp
```

主要 tools：

- `list_windows()` / `get_window_state(window_id)`
- `screenshot(pid, window_id)` — 5/2 後改成必填 window_id（避免抓整個桌面）
- `click(x,y)` / `type(text)` / `key(name)`
- `launch_app(name)`

**適合**：自己寫 orchestration loop，要能 swap LLM（Anthropic / OpenAI / 開源模型）。

### Path B — Claude Code computer-use 相容模式（PR [#1424](https://github.com/trycua/cua/pull/1424)，2026-05-02 merged）

如果 agent 後端是 Claude（用 native `computer_20241022` tool），不用自己 wrap MCP：

```bash
claude mcp add --transport stdio cua-computer-use -- cua-driver mcp --claude-code-computer-use-compat
```

- 保留正常 CuaDriver tools，只把 `screenshot` 換成 window-scoped 的 shim
- 暴露 `mcp__cua-computer-use__screenshot` 這個 tool name 作為 Claude Code 的 image-grounding cue
- 596 additions / 87 deletions / 20 files
- 配套有 cua-driver Skill：`libs/cua-driver/Skills/cua-driver/SKILL.md`

**適合**：agent 是 Claude based，最少黏合 code 直接跑。

### Path C — Sandbox / Lume（隔離環境）

cua-sandbox 跨 OS 管 VM/container lifecycle，Lume 在 Apple Silicon 上跑 macOS/Linux VM。

```bash
lume run macos-sequoia-vanilla:latest
```

**適合**：production 部署、大量平行 agent runs、RL 訓練、避免污染本機。

---

## Cua Driver 怎麼做到「不搶 focus」（macOS 內部機制）

cua-driver 30% 的 Swift code 是在處理「不搶游標 / 不切 frontmost / 不換 Space」這個問題。實際看了 source 後，技術重點：

### 1. 不走 HID tap，改 pid-routed delivery

macOS 一般合成事件（`CGEvent.post(tap: .cghidEventTap)`）走 WindowServer 全域 HID stream，會搬游標、搶 focus、改 frontmost。cua-driver 拒絕這條路，改用**直送目標 process mach port**，雙路並發：

| Path | API | 適用 |
|------|-----|------|
| 公開 API | `CGEvent.postToPid(pid)` | AppKit-style targets，SkyLight 對它常 silent drop |
| 私有 API | `SkyLightEventPost.postToPid`（auth-signed） | Chromium/Safari/Electron-family，AppKit 上不可靠 |

兩條路**互補**，driver 同時 fire 兩條，accept「事件可能到兩次」當代價（純 mach traffic，user 看不見）。**完全不動真實游標、不 activate 目標**。

### 2. 滑鼠：AX-first，pixel 路是 carve-out

```
有 AX？ → AXUIElementPerformAction（純 RPC，背景視窗也 work）
沒 AX？ → 合成 mouseDown/mouseUp pair @ pixel
        （Chromium 網頁 / Blender canvas / Figma / 遊戲）
```

非 AX 路最 tricky 的發現：**`NSEvent.mouseEvent(...).cgEvent` 走 AppKit bridge，比直接建 raw CGEvent 多一個「trust marker」**。Chromium renderer IPC layer 會把 raw-CGEvent 默默 filter 掉，但接受 NSEvent-bridge 的事件。Source comment 寫得很白：

> Switching to the NSEvent-bridged path fixed Chromium web-content hit-tests on backgrounded targets.

座標系手動翻 y（Cocoa bottom-left ↔ Quartz top-left），bridge 後再翻回來。`windowNumber` 故意傳 `0` — 傳真實 number 會被 HID-tap dispatcher reject。

### 3. 鍵盤：CGEvent + SkyLight auth envelope

- `CGEvent.postToPid(pid)` 直送 pid
- **SkyLight auth envelope** 附加在 event 上 — Chromium/Safari 背景時收 key 必須的 signed credential
- 對 native AppKit 的 **NSMenu key equivalents**（如 ⌘S 觸發選單項），auth envelope 反而會擋 → 改走 `IOHIDPostEvent`（PR #1437 的 fix）

### 4. 螢幕擷取：window-scoped capture

`CGWindowListCreateImage` / ScreenCaptureKit (`SCStream`) 抓特定 window image。視窗 minimize / 被遮擋 / 在另一個 Space / 完全 off-screen 都能抓 — 系統 Compositor 永遠維持每個視窗的 backing store。

### 5. 已知限制

> 像素路 right-click 進 Chromium 網頁內容會 land 成 left-click — Chromium 把非 HID-origin 的 pointer event coerce 回 primary button。

contextMenu 在網頁上要用 AX 路（`AXShowMenu`）才可靠。

### 6. Frontmost-only 例外

Blender、Unity、部分遊戲用 OpenGL/GHOST，**只接 HID stream**，pid-routed 進不去。fallback 到 `.cghidEventTap` + `mouseMoved`，代價搬游標。但只在目標已是 frontmost 才 trigger。

### 為什麼別人沒做到？

- `CGEvent.postToPid` 是公開 API 但很少人這樣用 — Apple 文件不會告訴你它不搬游標
- SkyLight 是私有 framework，要逆向 + auth-sign（trycua 投入不少時間 reverse-engineer）
- NSEvent-bridge trust 差異要遇到 Chromium 才會發現
- windowNumber 0、座標 flip、OpenGL fallback 都是踩坑得來

---

## Windows UIA 實作機制（Windows 對等方案）

UI Automation（UIA）是 Microsoft 從 Vista 開始推的 accessibility framework，**取代 MSAA**。對 agent 的價值：UIA 的 `InvokePattern.Invoke()` **直接呼叫 control 的 command handler**（透過 COM IPC 進到 target process），不是合成 input event — 所以**不搬游標、不搶 focus、不切 frontmost**，等同 macOS AX path。

### 1. 架構

```
[your agent process]                    [target app process]
     ↓                                          ↑
[UIAutomationCore.dll]  ←─── COM IPC ───→ [UIA Provider in target]
     ↓                                          ↑
[FlaUI / pywinauto / raw COM]            [內建 UIA support: WPF / WinUI / Win32]
```

- **Client side**（agent）：`UIAutomationCore.dll`，透過 COM 呼叫
- **Provider side**（target app）：app 本身要 expose UIA tree
  - WPF / UWP / WinUI：framework 自動產生（excellent）
  - Win32 common controls：built-in UIA bridge（OK）
  - Chromium / Electron：2020 後有 UIA support（部分 — 表單 / 按鈕 OK，canvas / WebGL 沒）
  - Qt / GTK / 自繪 GUI：差到沒
  - DirectX games：完全沒

### 2. 核心抽象

```
AutomationElement (UI tree node)
├── Name, AutomationId, ControlType, BoundingRectangle, Process ID
├── Patterns（可執行的介面，類似 Java interface）
│   ├── InvokePattern.Invoke()             — 按鈕、選單項、連結
│   ├── ValuePattern.SetValue("text")      — 輸入框直接灌值（不打字）
│   ├── TextPattern.GetText()              — 讀取 rich text 內容
│   ├── TogglePattern.Toggle()             — Checkbox
│   ├── SelectionItemPattern.Select()      — List / Tree item
│   ├── ExpandCollapsePattern.Expand()     — Tree node
│   └── WindowPattern.SetWindowVisualState — Minimize/Maximize/Close
└── TreeWalker / FindFirst / FindAll       — 找子元素
```

**關鍵**：你不是「點座標」，你是「找到 element → 呼叫它的 Pattern」。完全 logical layer，沒 input synthesis。

### 3. 三條實作路線

| 路線 | 語言 | 複雜度 | 推薦給 |
|------|------|--------|--------|
| **FlaUI** | C# / .NET | 低 | Windows-native agent，性能好 |
| **pywinauto** | Python | 低 | Python agent，原型快 |
| **Raw UIA COM** | C++ / Rust / Python via comtypes | 高 | 自己寫 driver，效能優化 |

### 4. 程式碼

**FlaUI（C#）— 推薦**

```csharp
using FlaUI.Core;
using FlaUI.UIA3;

var app = Application.Attach("notepad.exe");
using var automation = new UIA3Automation();
var window = app.GetMainWindow(automation);

var saveBtn = window.FindFirstDescendant(cf => cf.ByAutomationId("FileSaveButton"));
saveBtn.AsButton().Invoke();    // ← 不搶 focus

var textBox = window.FindFirstDescendant(cf =>
    cf.ByName("Username").And(cf.ByControlType(ControlType.Edit)));
textBox.AsTextBox().Text = "doro";   // SetValue，不打字
```

**pywinauto（Python）— 原型最快**

```python
from pywinauto import Application

app = Application(backend="uia").connect(title="Calculator")
window = app.window(title="Calculator")

window.print_control_identifiers()    # debug：列出 tree
window.child_window(title="Five", control_type="Button").invoke()
window.child_window(title="Plus", control_type="Button").invoke()
window.Edit.set_text("123")
```

**Raw UIA COM（Python via comtypes）— 效能極致**

```python
import comtypes.client
comtypes.client.GetModule('UIAutomationCore.dll')
from comtypes.gen.UIAutomationClient import (
    IUIAutomation, CUIAutomation,
    UIA_InvokePatternId, UIA_ButtonControlTypeId
)

uia = comtypes.client.CreateObject(CUIAutomation, interface=IUIAutomation)
root = uia.GetRootElement()

cond = uia.CreatePropertyCondition(UIA_ControlTypePropertyId, UIA_ButtonControlTypeId)
button = root.FindFirst(TreeScope_Descendants, cond)

ip = button.GetCurrentPattern(UIA_InvokePatternId)
ip.QueryInterface(IUIAutomationInvokePattern).Invoke()
```

### 5. 背景截圖（Windows 對應 `CGWindowListCreateImage`）

```python
import win32gui, win32ui
from ctypes import windll
from PIL import Image

hwnd = win32gui.FindWindow(None, "Visual Studio Code")
left, top, right, bot = win32gui.GetWindowRect(hwnd)
w, h = right-left, bot-top

hwnd_dc = win32gui.GetWindowDC(hwnd)
mfc_dc = win32ui.CreateDCFromHandle(hwnd_dc)
save_dc = mfc_dc.CreateCompatibleDC()
bmp = win32ui.CreateBitmap()
bmp.CreateCompatibleBitmap(mfc_dc, w, h)
save_dc.SelectObject(bmp)

# PW_RENDERFULLCONTENT = 3，關鍵：對 Chromium / DWM-composed 視窗有效
windll.user32.PrintWindow(hwnd, save_dc.GetSafeHdc(), 3)

img = Image.frombuffer('RGB', (w,h), bmp.GetBitmapBits(True), 'raw', 'BGRX', 0, 1)
```

**`PrintWindow` 重點**：
- `PW_RENDERFULLCONTENT (0x3)` 是 Win 8.1+ 加的，能抓 DWM composed surface（Chrome / Electron / 現代 app）
- 視窗 minimize / 被遮擋都能抓 — 跟 macOS Compositor 一樣維持 backing store
- 但完全 off-screen 的 child window 可能抓不到，沒 macOS 那麼乾淨

### 6. 效能 — caching 是必修

UIA 慢的原因是**每個 property access 都走 COM IPC**。Tree 大（Office、Chrome 多 tabs）走一遍要幾百 ms。

```csharp
// ❌ 慢：每個 property 一次 IPC
foreach (var e in elements) {
    Console.WriteLine(e.Name);          // IPC
    Console.WriteLine(e.AutomationId);  // IPC
}

// ✅ 快：批次預載
var cache = new CacheRequest();
cache.Add(AutomationElement.NameProperty);
cache.Add(AutomationElement.AutomationIdProperty);
cache.Add(InvokePattern.Pattern);
using (cache.Activate()) {
    var elements = root.FindAll(TreeScope.Descendants, condition);
    foreach (var e in elements) {
        Console.WriteLine(e.Cached.Name);    // 從 cache，0 IPC
        ((InvokePattern)e.GetCachedPattern(InvokePattern.Pattern)).Invoke();
    }
}
```

對 agent：每次 `screenshot` 後重新 walk tree（畫面變了）會炸效能。**好 design 是 element-id-based** — agent 第一次 query 拿到 ID list，後續 `invoke(elem_id)` 直接用 cached handle。pywinauto 預設不快取，自己包 wrapper 時要做。

### 7. 元素定位策略（穩定度）

```
最穩 ──────────────────────────────────────── 最脆
AutomationId → ControlType+Name → Name → BoundingRect → 像素座標
```

`AutomationId` 是 dev 自己設的（WPF `x:Name`、Win32 control ID），update UI 也不太會變。`Name` 容易因 i18n / 文案變動失效。

### 8. Inspect 工具（必裝）

- **Inspect.exe** — Windows SDK 內建，hover 在任何元素上看 UIA tree、property、可用 pattern。debug agent 第一個工具
- **Accessibility Insights for Windows** — Microsoft 開源，UI 比 Inspect 友善
- **FlaUInspect** — FlaUI 配套版，C# 開發者方便

寫 agent 第一步：用 Inspect.exe 看 target app 的 tree → 決定用哪個 pattern。

### 9. UIA support 實際狀況

| App 類型 | UIA 完整度 | 備註 |
|---------|-----------|------|
| WPF | 🟢 99% | framework 自動 expose，最理想 |
| WinUI 3 / UWP | 🟢 99% | 同上 |
| Win32（common controls） | 🟡 80% | UIA bridge 自動處理 |
| Office | 🟡 70% | UIA + 自家 ribbon UI；複雜 |
| Chromium / Electron | 🟡 60% | DOM 大致 OK，但 canvas / WebGL / 自繪 widget 沒 |
| VSCode / Slack（Electron-heavy） | 🔴 30% | 大量自繪，重要按鈕常找不到 |
| Java Swing / SWT | 🔴 0% | 有 Java Access Bridge 但要另裝 |
| Qt | 🟡 視 build 而定 | Qt 6 + accessibility plugin OK |
| DirectX / Unity / Unreal 遊戲 | 🔴 0% | 完全沒 |

**Electron-heavy app 是最大痛點**。VSCode 內 ribbon 按鈕找得到，但編輯區是 Monaco editor canvas，UIA 看不到 — 這時要 fallback 到 CDP（VSCode 有 debug protocol）或用 IPC（VSCode extension API）。

### 10. 包成 MCP server 的雛形

```python
# uia_mcp_server.py（pseudo-code）
from mcp.server import Server
from pywinauto import Application

server = Server("uia-driver")
elements_cache = {}  # id → element handle

@server.tool()
def list_windows():
    """List all top-level windows with HWND, title, pid."""
    return [{"hwnd": h, "title": t, "pid": p} for h,t,p in walk_top_windows()]

@server.tool()
def get_tree(hwnd: int, max_depth: int = 5):
    """Walk UIA tree of window, return element list with IDs."""
    app = Application(backend="uia").connect(handle=hwnd)
    window = app.window(handle=hwnd)
    elements = []
    for e in window.descendants():
        eid = id(e)
        elements_cache[eid] = e
        elements.append({
            "id": eid,
            "name": e.window_text(),
            "control_type": e.element_info.control_type,
            "automation_id": e.automation_id(),
            "rect": e.rectangle().as_tuple(),
            "patterns": list_patterns(e),
        })
    return elements

@server.tool()
def invoke(element_id: int):
    elements_cache[element_id].invoke()

@server.tool()
def set_value(element_id: int, value: str):
    elements_cache[element_id].set_text(value)

@server.tool()
def screenshot(hwnd: int):
    """Window-scoped screenshot via PrintWindow PW_RENDERFULLCONTENT."""
    return capture_window(hwnd)  # → PNG bytes
```

agent 流程：`list_windows` → `get_tree` 拿到 element list → LLM 看 list + screenshot → 決定 `invoke(id=42)` → `set_value(id=43, value="...")` → 再 `screenshot` 看結果。

### 11. UIA 不夠時的 fallback layers

| 場景 | Fallback |
|------|----------|
| Chromium 內網頁互動 | CDP（直接 attach Chrome debugger） |
| Electron canvas（VSCode Monaco） | IPC / extension API |
| 完全沒 UIA 的 legacy app | `PostMessage(WM_*)` 到 HWND |
| 上述都不行 | `SendInput` + frontmost（搬 focus，accept） |

**真實 production agent 通常是 5 條路 layered**：CDP → UIA → PostMessage → SendInput → 像素座標。每層失敗才 fallback 下一層。cua-driver 在 macOS 也是這個策略（AX → SkyLight → CGEvent.postToPid → cghidEventTap）。

---

## 跨平台替代方案

cua-driver 是 macOS-only 且要逆向 SkyLight 才寫得出來。Linux/Windows 上有更直接的路線。

### Linux：Docker + Xvfb

Linux + Xvfb 環境下「使用者根本不在」，cua-driver 30% 的工程在解的問題**直接消失**。

| CUA 元件 | macOS 上 | Linux + Docker + Xvfb |
|----------|----------|------------------------|
| 輸入合成 | `CGEvent.postToPid` + SkyLight + AX | `xdotool` / `xte` / pynput |
| 螢幕擷取 | `CGWindowListCreateImage` / SCStream | `xwd` / `scrot` / FFmpeg + X11 grab |
| 「不搶 focus」 | reverse-engineer SkyLight | **免費** — Xvfb 沒有真實使用者 |
| 隔離 | cua-sandbox | Docker 本身就是 |
| VM mgmt | Lume（AS-only） | 不需要（用 container） |
| 桌面串流 | H.265 | x11vnc + noVNC，或 FFmpeg → WebRTC |

**Linux 上「免費」拿到的**：
- xdotool 打事件 → X server → app，不必擔心搶 focus
- 所有 X11 事件都是 trusted，沒有 trust marker 差異
- 不會搬使用者游標（沒有真實游標）
- Chromium right-click coerce 那個 macOS bug 在 Linux 不會出現（X events 走同一條 protocol）

**注意事項**：
- **X11 only**：Xvfb 是 X server。Wayland-only app 走 XWayland 大致 OK
- **沒 GPU 加速**：軟體 framebuffer，OpenGL/WebGL/3D 跑不動。要 GPU 換 `Xorg` + nvidia-docker，或 `VirtualGL` + `Xvnc`
- **沒音訊**：要掛 PulseAudio / PipeWire socket
- **解析度固定**：啟動時鎖死，動態 resize 用 `Xvnc`
- **xdotool cursor**：預設搬 cursor，Xvfb 內 cursor 是 virtual 不影響使用者

**推薦 stack**：

```
[your Linux agent]
       ↓ MCP
[wrapper: xdotool + scrot/xwd + xwininfo]
       ↓ X11 socket
[Xvfb :99 inside Docker container]
       ↓
[target apps: chromium-headed, electron app, etc.]
```

**現成方案**：Selenium / Playwright + Chromium headed、PyAutoGUI in Xvfb、DesktopENV / OSWorld 的 Docker setup（學術界 computer-use benchmark 已這樣做）。

**Fork CUA 做 Linux port 可行性**：以 cua-sandbox 為核心加 `LinuxDockerXvfbDriver` plug-in，**比 macOS port 簡單得多** — 不用碰 SkyLight、不用 reverse engineer trust marker、不用處理「使用者在用電腦」。估 1–2 週做到 80% 功能。

### Windows：Playwright + UIA + pywinauto 混搭

> 實作細節見上「Windows UIA 實作機制」一節；本段聚焦 deployment / 技術選擇 trade-off。

Windows 跟 macOS 處境**像但更難**。沒有 SkyLight 等價物可逆向，Chromium 在 Windows 也 filter 掉非 HID 來源 input。

**Windows 上「不搶 focus」的選項**：

| 路線 | 對等於 macOS 哪條 | 不搶 focus？ | 限制 |
|------|------------------|-------------|------|
| **UI Automation `InvokePattern.Invoke()`** | AX path | ✅ Yes | 只 UIA-compliant app（modern Win32 / WPF / UWP）；Chromium 算半 UIA |
| **PostMessage(WM_LBUTTONDOWN, ...) 到 HWND** | `CGEvent.postToPid` | ✅ 不搶 focus | 多數現代 app（Chromium / Electron / 自繪 GUI / 遊戲）會 ignore HWND message |
| **SendInput** | `cghidEventTap` | ❌ Steals focus | 全域 HID stream，搬游標 |
| **DirectInput** | OpenGL/GHOST | N/A | 遊戲專用 |
| **WinAppDriver / Selenium** | — | ✅ | Microsoft 已 deprioritize；後端還是 UIA |

**關鍵 insight**：
- Chromium for Windows 用 Aura compositor + 自己 hook 全域 input pipeline，**PostMessage 對 Chrome 視窗等於沒效**
- Chromium 的解只有 **CDP（Chrome DevTools Protocol）**，沒有 OS-level 後門
- Native Win app 的解是 **UIA**，UIA-compliant 的 app 可以背景控制不搶 focus

**Windows 沒有 SkyLight 等價物**：cua-driver 在 macOS 突破的關鍵是 reverse-engineer SkyLight 的 auth-signed pid-routed event，Chromium 收。Windows 沒有對應私有 framework 可逆向 — Chrome 的 input filter 在 Aura compositor 內部，沒「跳過 HID 直送」的機制。

**推薦 Windows stack**：

```
[your agent]
   ├─ if target 是網頁              → Playwright/CDP（headless 或 Xvfb headed）
   ├─ if target 是 UIA-compliant    → pywinauto / FlaUI（背景）
   ├─ if target 是 legacy Win32     → PostMessage + UIA 混搭
   └─ if target 是遊戲 / 3D         → SendInput + frontmost（搬 focus，accept）
```

第一步直接用 **Playwright + pywinauto** 就 90% 涵蓋。

**為什麼不必 fork CUA 做 Windows port**：
- Microsoft 自己有 UIAutomation + WinAppDriver 生態
- 社群有 pywinauto / FlaUI / Inspect.exe，比 macOS 的 AX 工具鏈成熟
- Windows port 的 cua-driver 本質會變成「UIA wrapper + CDP shim」，差異化沒那麼大
- trycua 自己也是這個方向：sandbox 支援 `Image.windows()`，但 driver 沒做 Windows native 控制，靠 sandbox 內裝 Windows + 從外部用 RDP / VM tools

### 三平台對照總結

| 維度 | macOS | Linux + Xvfb | Windows |
|------|-------|--------------|---------|
| 「不搶 focus」 hard 嗎 | 🔴 hard，要 SkyLight | 🟢 free，沒人在用 | 🟡 medium，UIA 可，pixel 不行 |
| Chromium 控制 | 需 SkyLight + NSEvent bridge | xdotool 直接過 | **只能用 CDP** |
| Native app 控制 | AX + cua-driver | xdotool（少 native app） | UIA / pywinauto |
| GPU / 3D | OK（Metal） | ❌ Xvfb 無，要 GPU 容器 | OK（DirectX） |
| 工具鏈成熟度 | cua-driver 領先 | xdotool / Playwright | UIA 生態最成熟 |
| Production 部署 | 需要 Mac fleet（貴） | 容易（K8s + Docker） | 容易（Windows containers） |
| 自寫 driver 難度 | 🔴 高（逆向） | 🟢 低（成熟工具） | 🟡 中（API 多但分散） |

---

## 自寫 agent 的決策樹

```
你的 target 主要是？
├─ 網頁                          → Playwright / CDP，不必碰 cua（80% 場景）
├─ Mac 原生 app                  → cua-driver MCP（甜蜜點）
├─ Linux 原生 app                → 自寫 xdotool wrapper + MCP
├─ Windows 原生 app              → pywinauto / FlaUI + 自寫 MCP wrapper
├─ Windows + Chromium 混合       → Playwright（網頁部分）+ pywinauto（native 部分）
└─ 跨 OS desktop benchmark / RL  → cua-sandbox + Image.* VM 是合理選擇
```

**對 99% 的「我要做 SaaS agent」use case，headless Chrome + Playwright 就是答案**，cua 整個 stack 反而 over-engineering。

cua 的真正價值集中在：
1. **Mac native app 控制**（不搶 focus）
2. **Apple Silicon 上的 macOS / Linux VM lifecycle**
3. **Cross-OS computer-use benchmark / RL 訓練資料蒐集**
4. **research-grade 完整桌面 agent**（OSWorld / ScreenSpot / Windows Arena）

---

## 使用場景

- 建構自主桌面 Agent
- QA 自動化、RPA 替代方案
- Benchmark computer-use agent（OSWorld、ScreenSpot、Windows Arena）
- 在 GUI 互動上訓練 RL Agent
- 跨平台沙盒開發
- Apple Silicon 上跑近原生 macOS / Linux VM

---

## 評估

**優點**

- macOS native 背景控制（不搶 focus）是其他工具沒有的，Blender / Figma / DAW / game engines 等 canvas-based 工具也能 drive
- MCP 介面通用，agent 端 LLM 可自由切換
- 與 Claude Code 直接整合（PR #1424）→「整合 Anthropic Computer Use 生態」已落地
- Lume 在 Apple Silicon 上效能近原生
- 每個 session 自動錄成 replayable trajectory，方便 debug 與 RL 訓練資料蒐集

**注意**

- cua-driver / Lume 是 macOS / Apple Silicon 專屬
- Windows / Linux 沒「不搶 focus」對等實現；Linux 因 Xvfb 自然消失，Windows 要 UIA + CDP 混搭
- 12 天 5 個 release，API 可能還會 breaking — 短期內鎖版本
- **80% 的 agent 任務 headless Chrome 就解決，cua 是剩下 15–30% 的解**

**對自寫 agent 的選擇**

- Mac 開發、控制 Mac apps：直接 cua-driver MCP，最甜蜜點
- Mac 開發、控制 Windows guest：Mac host 跑 cua-sandbox + `Image.windows()` VM
- Linux server 部署：Docker + Xvfb + 自己包 MCP wrapper；或 cua-sandbox + container image
- Windows server 部署：Playwright + pywinauto，不需要 cua
- 真要 cross-OS production：cua-sandbox 為控制平面，driver 各自為政（macOS 用 cua-driver、Linux 用 xdotool、Windows 用 UIA + Playwright）
