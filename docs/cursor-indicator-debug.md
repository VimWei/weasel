# 光标位置 IME 状态图标调试记录

## 问题描述

使用 AppIME / im-control 自动切换 RIME/weasel 输入法的中英文状态时，**光标位置的语言栏按钮（`_pLangBarButton`）** 图标不更新。

例如：在 Windows Terminal 中编辑，AppIME 闲置 8 秒后自动切换为英文模式，右下角 RIME 托盘图标和任务栏"中"/"A" 指示器均正确显示"A"，但光标位置的语言栏按钮仍显示"中"。

RIME 托盘图标和任务栏"中"/"A" 指示器从未出问题，问题仅限于光标附近的语言栏按钮。

## 相关仓库

- `C:\Apps\git-kb\repos\VimWei\weasel\` — 小狼毫输入法（WeaselTSF 文本服务）
- `C:\Apps\git-kb\repos\VimWei\im-control\` — 输入法控制工具（hook 注入 + TSF compartment 写入）
- `C:\Apps\VimReader\lib\utils\IME.ahk` — AppIME 自动切换脚本
- `C:\Apps\git-kb\repos\VimWei\weasel\docs\cursor-indicator-debug.md` — 本文档
- `C:\Apps\git-kb\repos\VimWei\weasel\docs\archives\status-icon-debug.md` — 历史调试记录（术语混淆，保留原始状态）

## 测试日志

- `C:\Apps\git-kb\repos\VimWei\weasel\docs\VIMELNUC.log` — 2025-07-12 DebugView 捕获，Windows Terminal 约 44 秒操作

## 三个层级

| 层级 | 位置 | 数据源 | 更新方式 | 状态 |
|------|------|--------|---------|------|
| **RIME 引擎** | 右下角通知区域托盘图标 | RIME 内部 ascii_mode | WeaselServer IPC → 更新托盘图标 | ✅ 始终正确 |
| **任务栏"中"/"A" 指示器** | 任务栏右侧系统托盘区 | `_status.ascii_mode` | `_SetCompartmentDWORD()` 写 TSF compartment → TSF 渲染 | ✅ 始终正确 |
| **语言栏按钮 (`_pLangBarButton`)** | 光标附近的系统语言栏按钮 | `_status.ascii_mode`（同上） | `_pLangBarButton->UpdateWeaselStatus()` → TSF OnUpdate | ❌ 唯一问题 |

## 当前问题

| 场景 | 光标语言栏按钮 |
|------|---------------|
| 手动 Shift | ✅ |
| gvim（Win10/Win11） | ✅ |
| AppIME（Total Commander/Windows Terminal） | ❌ 显示"中" |

### Windows Terminal 执行指令时的特殊现象

在 Windows Terminal 中执行一条指令，如果该指令恰好切换 RIME 语言状态（如 AppIME 自动切换），光标位置会发生变化：
- **旧光标位置** → 语言栏按钮显示"中"（未刷新）
- **新光标位置** → 语言栏按钮显示"A"（已更新）

说明 `UpdateWeaselStatus` 只刷新**当前活跃文本上下文**的语言栏按钮，旧位置（已失焦）的上下文不接收更新。

## 2025-07-12 日志验证结果

从 `VIMELNUC.log`（DebugView 捕获，Windows Terminal 约 44 秒操作）分析：

### 确认的结论

1. **✅ OnChange(CONVERSION) 跨线程触发** — 1 次外部写事件（`_updatingLangBar=0`）在 t=29.62s 触发，确认 im-control 在 hook 线程写 compartment 后，TSF 会在该线程的 WeaselTSF 实例上触发 OnChange
2. **✅ `_pLangBarButton` 非空** — 3 个线程的 LBB 均非 NULL，0 次 NULL 事件，`UpdateWeaselStatus` 的条件不会被空指针阻挡
3. **✅ `_ReconcileCompartment` 无异常** — 全程 0 次 MISMATCH，`_status` 与 compartment 始终一致

### 发现的根因：TSF compartment 是 per-thread 的

日志中出现了 **3 个线程**，各有独立的 TSF compartment 和 `_pLangBarButton`：

| Thread | _status 变化 | 收到 OnChange 外部触发 |
|--------|-------------|----------------------|
| 8868 | 始终 0（中） | ❌ 从未 |
| 9332 | 始终 1（英） | ❌ 从未 |
| 12864 | 0 → 1（t=29.62 切换） | ✅ 1 次 |

核心发现：**TSF 的 `GUID_COMPARTMENT_KEYBOARD_INPUTMODE_CONVERSION` 是 per-thread-manager 的**。im-control 写入 compartment 只影响调用线程（12864）的 compartment，其他线程（8868、9332）的 compartment 保持不变。`_ReconcileCompartment` 在 8868 上比较 `_status=0` vs `compartment=0`，认为没有 mismatch，所以不会调用 `UpdateWeaselStatus`。

**为什么任务栏"A"但光标"中"？**
- **任务栏"A"** ← 前台窗口线程（12864）的 compartment=1 ✅
- **光标"中"** ← 活跃文本上下文所在线程（8868）的 compartment=0，`_pLangBarButton` 从未收到更新 ❌

**im-control 的 hook 执行流程**：`main.cpp` 通过 `GetForegroundWindow()` + `GetWindowThreadProcessId()` 获取前台窗口线程 → `SetWindowsHookEx(WH_CALLWNDPROC, ..., dwThreadId)` 注入到该线程 → `hook.cpp` 调用 `TF_GetThreadMgr()` 获取 per-thread 单例 → 写 compartment。但前台窗口的线程可能与活跃 TSF 文本上下文的线程不是同一个（Windows Terminal 多线程架构）。

## 可能的解决方向

### 方向 A：im-control 写入所有线程的 compartment

在 im-control 端，写入 compartment 后，枚举目标进程的所有线程，对每个线程调用 `TF_GetThreadMgr()` 获取其 ThreadMgr，重复写入相同值。确保所有 per-thread compartment 一致。

**优点**：从源头根治，每个线程的 compartment 都能同步。
**缺点**：跨线程注入复杂度高，需要处理线程同步和竞争条件。

### 方向 B：im-control 通知目标窗口广播

im-control 写完 compartment 后，使用 `RegisterWindowMessage` + `SendMessage`（`HWND_BROADCAST` 或枚举目标进程窗口）发送自定义消息。WeaselTSF 的 `_DeferredWndProc` 处理该消息，执行 `_ReconcileCompartment` + `_UpdateLanguageBar`。

**优点**：轻量，不需要改 WeaselTSF 定时器逻辑。
**缺点**：仍需要 per-thread 消息泵活跃才能收到消息；广播可能影响无关窗口。

## Debug 日志已添加

| 位置 | 前缀 | 用途 |
|------|------|------|
| `_HandleCompartment(OPENCLOSE)` 入口 | `WTSF_OnChange(OPENCLOSE)` | 确认 open/close 触发 |
| `_HandleCompartment(CONVERSION)` 入口 | `WTSF_OnChange(CONVERSION)` | 确认 conversion 跨线程触发 |
| `_ReconcileCompartment` | `WTSF_Reconcile` | 检查 `_pLangBarButton` 状态和 mismatch |
