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

## 关键补充事实（2026-07-12 二轮代码复核）

原根因分析正确，但漏掉两个关键事实，影响修复方案：

### 补充事实甲：每条线程是独立 RIME session

每个 WeaselTSF 实例的 `m_client` 与 WeaselServer 之间是**独立 session**。
`ClientImpl::TrayCommand(menuId)` 携带 `session_id` 作 lParam → ServerImpl::OnCommand → `RimeWithWeaselHandler::SetOption(lParam, ...)` → 命中 else 分支 → **只切当前 session 的 ascii_mode**。

→ 12864 上的 `_HandleCompartment(CONVERSION)` 调用 `_HandleLangBarMenuSelect(ID_WEASELTRAY_ENABLE_ASCII)` 只切 12864 自己的 RIME session。8868/9332 的 RIME 后端状态不会跟随切换。

### 补充事实乙：WeaselServer /ascii 不保证广播

AppIME 在写完 compartment 后调用 `WeaselServer.exe /ascii`（IME.ahk）。这是无 session 客户端（`ipc_id=0`）。`RimeWithWeaselHandler::SetOption` 的 `if (!ipc_id)` 分支：**仅当 `global_ascii` 配置为 true 时**才广播到全部 session（RimeWithWeasel.cpp:494-496）。

→ 默认配置下不能用 `WeaselServer /ascii` 兜底 RIME 后端同步。

### 修复必须同时解决的三件事

| # | 要解决的同步 | 只刷 LBB 不切 session 的后果 |
|---|-------------|----------------------|
| 1 | 8868/9332 的 `_status.ascii_mode` | 0 → 下次按键被响应器改回 |
| 2 | 8868/9332 的 `_pLangBarButton` 图标 | 仍是"中"（当前问题） |
| 3 | 8868/9332 自己 RIME session 的 ascii_mode | 仍是中文 → 下次按键 `_UpdateLanguageBar` 把 compartment 回写为 0、LBB 被静默翻回"中"（与归档文档"撤销外部变更"旧 bug 同型） |

只刷 2 不刷 3 = 在新层面复现"撤销外部变更"旧 bug。明确拒绝。

## 实施方案：方向 C —— Weasel 进程内广播（不动 im-control）

让 12864 的 `_HandleCompartment(CONVERSION)` 检测到外部驱动的 ascii 翻转后，向同进程其他 WeaselTSF 实例的 `_hDeferredMsgWnd` 投递自定义消息 `WM_APP+101`，由对方在自己线程的消息泵里原样重放"切换"动作。1/2/3 三件事一次解决。

### 复用现成零件

- `_hDeferredMsgWnd`（WeaselTSF.cpp ActivateEx 段）：每个实例已有一个 message-only 隐藏窗口
- `_DeferredWndProc`（WeaselTSF.cpp）：原有 `WM_APP+100` 分支，新增 `WM_APP+101`（`WM_TIMER` 分支已删除，见下文"定时器决策"）
- `g_cs`（Globals.cpp）：现成全局 CRITICAL_SECTION，作实例注册表锁
- `_HandleLangBarMenuSelect(ID_WEASELTRAY_*_ASCII)` + `_UpdateLanguageBar(_status)`：CONVERSION handler 末尾的现成动作，接收方按原顺序复用

### 改动文件清单

| 文件 | 改动 |
|------|------|
| `WeaselTSF/Globals.h` | 新增 `WeaselTSF*` 实例注册表声明与访问函数 |
| `WeaselTSF/Globals.cpp` | 实例注册表实现，复用 `g_cs` |
| `WeaselTSF/WeaselTSF.h` | 暴露 `_GetDeferredWnd()`；新增 `_OnRemoteAsciiChange(bool)` |
| `WeaselTSF/WeaselTSF.cpp` | `ActivateEx/Deactivate` 注册/注销；`_DeferredWndProc` 新增 `WM_APP+101`；新增 `_OnRemoteAsciiChange`；**移除 2s `SetTimer` 与 `WM_TIMER` 分支** |
| `WeaselTSF/Compartment.cpp` | `_HandleCompartment(CONVERSION)` 外部驱动翻转分支末尾向其他实例 `PostMessage(WM_APP+101, ascii, 0)` |

### 安全性原则（与归档 compartment-external-control-fix.md 同）

- **OPENCLOSE handler 零改动**：Shift / Ctrl+Space 主功能不受影响
- **不动 `_isToOpenClose` 分歧**：广播的是目标 ascii 值；OPENCLOSE 维度处理逻辑未触
- **幂等三重保险**：
  - 发起方：`_updatingLanguageBar` 守卫保留；本线程 OnChange 回路命中 else 不动作
  - 接收方：`if (_status.ascii_mode == ascii) return;` 早退
  - 接收方写本线程 compartment 后，本线程 OnChange 同样命中 else 早退
- **跨线程安全**：所有 RIME session 调用、compartment 读写、LBB UI 调用在对方线程的消息泵里执行，无锁、无重入。注册表 `g_cs` 仅在 `push/erase/snapshot` 瞬间持锁
- **不动 im-control / 不动 WeaselServer IPC**：与现有 AppIME 流程完全兼容
- **deactivate 安全**：`Deactivate` 先 `Weasel_UnregisterInstance(this)` 再销毁窗口，避免 use-after-free

### 定时器决策

**去掉 2s `SetTimer` 与 `WM_TIMER` 分支**，只靠广播。broadcast-first：
- `_ReconcileCompartment` 函数仍保留（`OnSetThreadFocus` 还在调用），只是不再被定时器周期触发
- 保留 `WM_APP+100` 备路（`_HandleCompartment` 末段仍会 Post 给自己），属于同步路径的兜底，与 `WM_APP+101` 跨线程广播互补

### 接收方动作时序（_OnRemoteAsciiChange）

与发起方 `_HandleCompartment(CONVERSION)` 外部驱动分支同顺序，幂等早退为先：

1. 调试日志，`if (_status.ascii_mode == ascii) return;` 幂等早退
2. `_status.ascii_mode = ascii;`
3. `if (_isToOpenClose && !_IsKeyboardOpen()) _SetKeyboardOpen(true);` 保底重开键盘（与归档修复"_isToOpenClose=true 时双重保底"原则一致）
4. `if (_pLangBarButton && _pLangBarButton->IsLangBarDisabled()) _EnableLanguageBar(true);`
5. `_HandleLangBarMenuSelect(...)` 通知本线程 RIME session 切 ascii（解决补充事实甲/乙）
6. `if (_pEditSessionContext) m_client.ClearComposition();`
7. `_UpdateLanguageBar(_status)`：内部按 `_status` 调整本线程 compartment flags、`_updatingLanguageBar=true` 守卫下写 compartment、再 `_pLangBarButton->UpdateWeaselStatus(_status)` 刷图标。本线程 OnChange 在守卫期返回 S_OK，守卫结束后命中 else 不动作 → 不递归、不再广播

### 发起方动作时序（_HandleCompartment CONVERSION 外部驱动分支末尾）

在已有 `_pLangBarButton->UpdateWeaselStatus(_status);` 之后：

```cpp
auto snap = Weasel_SnapshotInstances();
for (auto* other : snap) {
  if (other == this) continue;            // 自己刚刷完
  if (HWND h = other->_GetDeferredWnd())
    PostMessage(h, WM_APP + 101, (WPARAM)(desiredAsciiMode ? 1 : 0), 0);
}
```

snapshot 持锁瞬取，循环不持锁，`PostMessage` 异步，对已 `Deactivate` 的实例安全（其窗口先于注销被销毁，不在 snapshot 里）。

## 验证计划

1. 编译：`build.bat weasel release` → `output\weaselx64.dll`、`output\weasel.dll`
2. 部署：管理员 PowerShell 覆盖 `C:\Windows\system32\weasel.dll` 与 `C:\Windows\SysWOW64\weasel.dll`（先 `.bak`）
3. 重启 Windows Terminal / Total Commander / gvim 等 RIME 宿主进程（必须重载 DLL）
4. 复现场景：AppIME 8s 自动切英文
   - 期望：托盘"A"、任务栏"A"、**光标附近 LBB 立即"A"**，无需先按键；新旧光标位置都正确
5. DebugView 关键日志：新增 `WTSF_Broadcast: ascii=1 self=12864` 以及每个接收线程的 `WTSF_OnRemoteAscii`
6. 主功能回归：手动 Shift、Ctrl+Space、gvim `i`/`ESC`、Windows Terminal 焦点进出均正常

## 拒绝的备选方案

- **方向 A（im-control 枚举全部线程写 compartment）**：能补 compartment 同步，但解决不了"接收方 RIME session 仍未切"，且跨线程注入复杂、风险高
- **方向 B（im-control 广播自定义窗口消息）**：message-only 窗口收不到顶层广播，WeaselTSF 现成无任何自定义消息入口，新增面比方向 C 大
- **只刷 LBB 不切 RIME session**：等价于把归档修复"撤销外部变更"旧 bug 在新层面复现，明确拒绝

## 为什么 vim-im-select 没有这个问题

vim-im-select (`C:\Apps\vim-init\pack\mydev\opt\vim-im-select\`) 与 AppIME (`C:\Apps\VimReader\lib\utils\IME.ahk`) 走完全一样的 im-control 通路 —— 都是 `im-control -c alphanumeric/native` + `WeaselServer.exe /ascii` 两次调用。但 vim-im-select 不存在"光标 LBB 不刷新"的问题,核心区别**不在调用方式,在触发时机**。

### vim-im-select 触发点全都伴随 TSF 事件

`plugin/im_select.vim` 注册的 autocmd:

| Vim autocmd | 同时发生的 TSF 事件 | 已有 handler 已会刷新 LBB |
|------------|---------------------|---------------------------|
| `InsertEnter` / `CmdLineEnter` | 用户刚按下 `i/a/o/:` 键 → `WeaselTSF::OnKeyDown` | `m_client.ProcessKeyEvent(ke)` 拉 server 权威态 → `ResponseParser` 写 `_status.ascii_mode` → `_UpdateLanguageBar(_status)` 写本线程 compartment + 调 `_pLangBarButton->UpdateWeaselStatus()` |
| `InsertLeave` / `CmdLineLeave` | 用户按 `Esc` → 同上 | 同上 |
| `FocusGained` / `FocusLost` | vim 窗口拿焦点 → `WeaselTSF::OnSetThreadFocus` | `m_client.ProcessKeyEvent(0)` + `GetResponseData(parser)` → 同上 |
| `TermEnter` / `TermLeave` | 终端焦点切换 → OnSetThreadFocus 类似 | 同上 |

每次切换都紧贴着用户的一个按键或一次焦点切换,**用户的下一条 keystroke 立刻会经过 `WeaselTSF::OnKeyDown`,从 server 拉回权威 `status.ascii_mode`,`_UpdateLanguageBar` 顺手刷 LBB**。被切的是英文模式 → 用户接下来打英文字符 → OnKeyDown → server 返回 ascii=true → LBB 立即"A"。延迟 < 一个 keystroke,用户感知不到。

### AppIME 触发点是闲置定时器,没有任何伴随 TSF 事件

`IME.ahk` 是 AutoHotkey 后台脚本,触发时机是"闲置 8 秒"。这时:

- 用户**没有按键** → 没有 `WeaselTSF::OnKeyDown` → 不会调 `ProcessKeyEvent` 拉 server
- 用户**没有切换焦点** → 没有 `OnSetThreadFocus` → 不跑 `_ReconcileCompartment` + `_UpdateLanguageBar`
- 客户端 WeaselTSF 实例完全"睡着"

`im-control -c alphanumeric` 只能写前台窗口线程的 per-thread compartment(`GetForegroundWindow()` + `SetWindowsHookEx(WH_CALLWNDPROC, ..., dwThreadId)`),光标的 TSF 文本上下文很可能在另一个进程(WindowsTerminal.exe vs OpenConsole.exe/conhost.exe 子进程)。`WeaselServer /ascii` 把 server 端所有 RIME session 的 ascii_mode 拉为 true,但客户端 LBB 只在下次 `ProcessKeyEvent` 回包时才同步——闲置中根本没有下次按键。结果:server 端权威态已切,客户端 LBB 没人触发 `UpdateWeaselStatus`,停留在"中"。

### 一句话总结

| 工具 | 触发时机 | 是否伴随 TSF 事件 | 谁负责把 server 权威态拉回 client LBB |
|------|---------|------------------|--------------------------------------|
| vim-im-select | 用户键入 `i/Esc/FocusGained` 等关键事件时 | 是 | 用户下一个 keystroke 自然触发 `WeaselTSF::OnKeyDown` → 从 server 拉 → 刷 LBB |
| AppIME | 闲置 8 秒定时器 | 否 | 无人触发,需要靠本轮新增的 2 秒定时器主动拉 server 才能刷 |

这也解释了为什么本轮修复要重加 2 秒定时器:它在 AppIME 闲置场景下**扮演 vim-im-select 上下文中"用户的下一次按键"这个角色**,强行让客户端周期性从 server 拉权威态刷 LBB。

## 首次实施（方向 C）部署后实测：失败

部署 commit `8d3f406` 后测试：AppIME 闲置切换时**光标处状态图标依然显示"中"**，托盘/任务栏正确。

### 日志（部署后）

`docs/debugview/VIMELNUC02.log`（2026-07-12 二次部署后捕获）：

```
t=0.00   PID 4504  OnChange(_updating=1 asc=0)         ← 4504 自写
t=10.10  PID 4504  Reconcile(_status=0 comp=0)          ← 4504 焦点切换
t=12.41  PID 8784  Reconcile(_status=1 comp=1)          ← 8784 焦点切换
t=12.41  PID 8784  OnChange(_updating=1 asc=1)           ← _UpdateLanguageBar(asc=1)
t=14.48  PID 8784  OnChange(_updating=1 asc=1)
t=14.59  PID 8784  OnChange(_updating=1 asc=0)          ← _UpdateLanguageBar(asc=0)
t=23.07  PID 8784  OnChange(_updating=0 asc=0)          ← 外部 compartment 写入
t=23.07  PID 8784  WTSF_Broadcast: ascii=1 self=10168   ← 进入广播分支（self=TID 10168）
t=35.44  PID 4504  Reconcile(_status=0 comp=0)          ← 4504 仍 _status=0
```

### 新诊断：跨进程，不是跨线程

DebugView 第三列为 PID（含 `self=` 行给出 TID 区分提示）。4504 / 8784 / 10168 不是同进程不同线程，而是**承载 IME 的多个进程**：Windows Terminal 与 conhost/OpenConsole 各自加载一份 `weasel.dll`，每个进程维护各自的 `g_weaselInstances`。

进程内实例注册表 + `PostMessage(WM_APP+101)` 只能命中同进程其它线程的 WeaselTSF。光标所在的 LBB 若长在另一进程里，永远不会被本次广播触及。日志中 4504 全程没有 `WTSF_OnRemoteAscii`，证实跨进程同步缺失。

> 注：上次决定"去掉定时器，只靠广播"基于"广播能覆盖所有线程"的假设。实测推翻——单进程内广播够不到其它进程的 WeaselTSF 实例。需要换路径。

## 二次实施：Server 广播 + 客户端 2 秒周期拉取权威态

借道 WeaselServer——每个 WeaselTSF 的 `m_client` 已经与 server 维持独立 pipe session，server 是所有 session 的 hub。配合两条改动即可跨进程同步所有 cursor LBB：

### 改动 A（Server）：`SetOption(0, "ascii_mode", val)` 一律广播到全部 session

**文件：** `RimeWithWeasel/RimeWithWeasel.cpp` 的 `RimeWithWeaselHandler::SetOption`。

原代码：

```cpp
if (!ipc_id) {
  if (m_global_ascii_mode && opt == "ascii_mode") {
    for (auto& pair : m_session_status_map)
      rime_api->set_option(to_session_id(pair.first), "ascii_mode", val);
  } else {
    rime_api->set_option(to_session_id(m_active_session), opt.c_str(), val);
  }
}
```

改：`opt == "ascii_mode"` 的无 session 写入**一律**广播到全部 session，不再依赖 `m_global_ascii_mode`。AppIME 已在每次 compartment 写后调用 `WeaselServer.exe /ascii`，会经此路径广播到所有 RIME session 的 ascii_mode 服务端权威态。

### 改动 B（Client）：2 秒周期 timer 拉 server 权威态，刷本地 LBB

**文件：** `WeaselTSF/WeaselTSF.cpp`。重加 `_InitDeferredWindow` 的 `SetTimer(hWnd, 1, 2000, NULL)`、`_UninitDeferredWindow` 的 `KillTimer`、`_DeferredWndProc` 的 `WM_TIMER` 分支——但**语义不再是本地 compartment reconciliation**，而是仿照 `OnSetThreadFocus` 的"握手"：

```cpp
if (m_client.Echo()) {
  m_client.ProcessKeyEvent(0);
  weasel::ResponseParser parser(NULL, NULL, &_status, NULL, &_cand->style());
  m_client.GetResponseData(std::ref(parser));
}
_UpdateLanguageBar(_status);
```

每个 WeaselTSF 实例无论属于哪个进程，只要有 thread focus 都会每 2 秒向 server 拉一次权威态，写本地 compartment + 刷 LBB。失焦的旧 cursor 线程在没 focus 时不会刷——但只要 server 已通过 A 把该 session 切成 ascii，下次它拿到 focus 或下个按键就会立刻 sync。最坏有 2 秒延迟（focus 间隔的 LBB 残影几乎不可见，因为派给 LBB 渲染的就是当前 focus 线程）。

新增私有方法 `_RefreshStatusFromServer()` 封装上述握手序列。

### 改动 C：保留上次的进程内广播

不撤回方向 C 的 5 文件改动——同进程多线程场景（如 conhost 前台 thread + ME 自身 thread）仍能受益，与改动 B 叠加不冲突（幂等）。

### 改动文件清单（v2）

| 文件 | 改动 |
|------|------|
| `RimeWithWeasel/RimeWithWeasel.cpp` | `SetOption(0, "ascii_mode", val)` 一律广播 |
| `WeaselTSF/WeaselTSF.h` | 新增私有 `void _RefreshStatusFromServer();` |
| `WeaselTSF/WeaselTSF.cpp` | 重加 SetTimer/KillTimer；`_DeferredWndProc` 的 `WM_TIMER` 改调 `_RefreshStatusFromServer`；实现新方法 |

### 安全性原则

- **OPENCLOSE handler / `_isToOpenClose` 分歧完全不受影响**
- **`global_ascii` 用户配置的语义不变**：只放宽无 session 调用对 ascii_mode 的广播。其余 option 仍按原逻辑只切 active session 或读 global_ascii
- **客户端 timer 是 server 权威态拉取**而非本地 reconcilation——`_UpdateLanguageBar` 已有 `_updatingLanguageBar` 防递归守卫，`ProcessKeyEvent(0)` 是 no-op 不入队字符。幂等
- **AppIME 调用流程不变**：`im-control -c alphanumeric` + `WeaselServer /ascii` 保持原样，行为只变好不变坏
- **per-process 进程内广播（v1 残留）与跨进程 server 拉取叠加**：两路互相补漏，互不破坏（幂等）

### 验证计划（v2）

1. 编译：`build.bat weasel release`
2. 部署：覆盖 system32 / SysWOW64 weasel.dll，**同时部署新编译的 WeaselServer.exe**（本次改了 RimeWithWeasel 编进 server 端）
3. 重启所有 RIME 宿主进程；如可重启 WeaselServer.exe
4. 复现场景：AppIME 8s 自动切英文
   - 期望：光标处 LBB 在 ≤2 秒内同步显示"A"
5. DebugView 关键日志：每 2 秒应见各 PID 的 `WTSF_RefreshFromServer`（待加），server 端 `_Respond` 内按 `global_ascii` 已存在的 `set_option` 广播路径命中所有 session
6. 主功能回归：手动 Shift、Ctrl+Space、gvim i/ESC、Windows Terminal 焦点进出均正常；不做手动切换时无异常刷新（_UpdateLanguageBar 比对 _status 与 server 一致)
7. 性能观察：每 2 秒/实例的 no-op ProcessKeyEvent(0) 流量可接受（< 4 字节 pipe transact * 3 个进程 = 总 ~12B/2s）

## 三次实施（v2 部署后）：2s 定时器实测能修复但架构 ugly，绑回 v3 改造 AppME 触发机制

v2（commit 31b914d）部署后实测：光标 LBB 在 ≤2 秒内同步显示"A"，问题修复。但 architecturally ugly：

- 每个 WeaselTSF 实例**每 2 秒**主动发起一次 `m_client.ProcessKeyEvent(0)` + `GetResponseData` 与 WeaselServer 的 pipe 通信
- 三个进程同时活 → 3 个 WeaselServer 端线程每 2s 唤醒一次，参与 g_api_mutex 串行化
- 即便没有外部切换，定时器也在持续打 tick → CPU 空 burn + 不必要的 IPC round-trip

用户提出更优雅的方向：**改造 AppIME，闲置时直接发送一次 Shift 按键让 RIME 自然切换**，与用户手动按 Shift 完全等价的路径——不再需要 WeaselTSF 端做任何轮询/广播兜底。

### v3 核心洞察：Shift path 已经包含 LBB 刷新

按键路径已确认天然刷新 LBB（`EditSession.cpp:6-16` 的 `DoEditSession`）：

```cpp
STDAPI WeaselTSF::DoEditSession(TfEditCookie ec) {
  // ... m_client.GetResponseData(parser);  ← 拉 server 当前状态
  _UpdateLanguageBar(_status);  ← 写本线程 compartment + UpdateWeaselStatus 刷 LBB
  // ...
}
```

每次 `WeaselTSF::OnKeyDown/OnKeyUp` → `_UpdateComposition(pContext)` → `RequestEditSession(this)` → `DoEditSession` 跑一遍：服务端响应 → 解析 `_status.ascii_mode` → `_UpdateLanguageBar(_status)` → `_pLangBarButton->UpdateWeaselStatus(_status)` 刷光标 LBB。

也就是说**只要让 AppIME 模拟用户按一次 Shift**，前台 WeaselTSF 的 OnKeyDown→OnKeyUp 路径会：
1. 把 Shift 按键发给 server `ProcessKeyEvent`
2. 服务端 RIME `ascii_composer` 检测到 lone Shift release → toggle `ascii_mode` Chinese↔English
3. 响应回包包含新 `status.ascii_mode` → `_status` 更新 → `_UpdateLanguageBar` 刷 LBB

这恰好就是 vim-im-select 用户每次手动按 `i/Esc` 时的同一条路径，无需任何 WeaselTSF 端的 workaround。

### v3 改动文件清单

| 端 | 文件 | 改动 |
|------|------|------|
| AppME | `C:\Apps\VimReader\lib\utils\IME.ahk` | 新增 `IME_SetEnglishViaShift()`，用 `Send("{LShift down}") Sleep(10) Send("{LShift up}")` 模拟用户按 Shift |
| AppME | `C:\Apps\VimReader\lib\system\AppIME.ahk` | 两处触发点（进入目标窗口 + 闲置 8s 切换）都改为调用 `IME_SetEnglishViaShift()` |
| Weasel | `WeaselTSF/WeaselTSF.cpp` | **回退** v2 加入的 `SetTimer / KillTimer / WM_TIMER 分支 / _RefreshStatusFromServer` |
| Weasel | `WeaselTSF/WeaselTSF.h` | **回退** `void _RefreshStatusFromServer();` 私有方法声明 |
| Weasel | `RimeWithWeasel/RimeWithWeasel.cpp` | **回退** v2 对 `SetOption(0, "ascii_mode", ...)` 的广播改动，恢复 `if (m_global_ascii_mode && opt == "ascii_mode")` 原条件 |

### 保留

- **commit 8d3f406 的进程内广播**（Globals.h/cpp 的 `Weasel_*Instance` + `Compartment.cpp` 的 `WM_APP+101` 广播 + `_OnRemoteAsciiChange`）。该机制作为 defense-in-depth 保留——在没有外部 compartment 写入时是死代码不触发；若将来有其他工具（如 im-control 的 `-c` 调用）仍写前台线程 compartment，该路径仍能帮助同进程多线程同步。
- **WeaselServer.exe /ascii** 入口（其他工具可能仍调用它，原 `global_ascii` gating 不变）。

### 触发时机/条件

AppME 既有的"`IME_GetMode()` 检测当前状态"逻辑保留：

- **进入目标窗口**：检测若 `mode == "close" || mode == "native"`（中文/关闭），对 `close` 先 `IME_OpenKeyboard` 重开键盘，再 `IME_SetEnglishViaShift()` 发一次 Shift 切英文
- **闲置 8 秒后**：检测若 `mode == "native"`（中文）才 `IME_SetEnglishViaShift()` 发 Shift 切英文

**关键**：状态检测先行保证我们只在"当前中文"时发 Shift。RIME 的 Shift release 是无方向 toggle——若已英文时发 Shift 会被切回中文。AppME 的 `if (mode = "native")` 守卫正好避免这个反向切换。

### AES 安全性原则

- **OPENCLOSE handler / `_isToOpenClose` 分歧完全未触**：与 v1/v2 同
- **WeaselTSF C++ 改动归零**：v2 客户端 inconvenient 全部回退，专有 helper GLFW/IPC 都不再新增。WeaselTSF 现状等价于 commit `8d3f406` 后的状态
- **RIME 状态变更走 process_key 自然路径**：与用户手动按 Shift 完全一致；任何其他 app 也调不出副作用
- **AppME 状态不变**：原有的 `IME_GetMode` / `IME_SetAlphanumeric` / `IME_SetNative` / `IME_OpenKeyboard` / `IME_SetEnglish` / `IME_SetChinese` 全部保留，其他 caller（如 `VimSimulator/SingleKeyMode.ahk`）不受影响
- **directional safety**：AppME 触发点都用 `IME_GetMode()` 先验证当前为 native 再发 Shift，不会反向 toggle

### v3 验证计划

1. 重新部署 v3 weasel.dll（system32 + SysWOW64），重启所有 RIME 宿主进程
   - **不需要重新部署 WeaselServer.exe**（v3 已回退 RimeWithWeasel，等价于 v0 的 server 状态；除非你部署过 v2 的 server，那就允许保留 v2 server，因为 v3 client 不依赖 server 端的广播改动——RIME Shift toggle 自身就会走 `process_key` 并自然更新 service 端 session 状态）
2. 重新加载 AppME AHK 脚本，让 `IME_SetEnglishViaShift()` 生效
3. 复现场景：AppME 8s 闲置切英文
   - 期望：光标 LBB **立即**显示"A"（伴随 Shift keystroke 完成而刷新，无延迟）
4. DebugView 关键日志：**无**任何 `WTSF_RefreshFromServer`、`WTSF_Broadcast`；**有**前台进程的 `WTSF_OnChange(_updating=1 asc=1)`（因 `_UpdateLanguageBar` 在守卫期间写 compartment）
5. 主功能回归：手动按 Shift 切换、Ctrl+Space、gvim `i`/`ESC`、Windows Terminal 焦点切换均正常
6. 副作用观察：进入目标窗口瞬间 AppME 可能发送一次 Shift，确认目标窗口在该瞬间没有"按住 Shift 左键 + 字符" 之类的 modifier 行为（WindowsTerminal / TotalCommander / IrfanView / gvim / mpv 通常都没有 Shift-alone 快捷键）

### v2/v3 对比

| 项 | v2 (commit 31b914d) | v3 (本轮) |
|----|---------------------|-----------|
| 同步机理 | 客户端 2s 定时器主动 poll server 权威态 + server 端 `/ascii` 一律 broadcast | AppME 模拟一次 Shift keystroke，走 RIME natural toggle path |
| WeaselTSF C++ 改动 | 加 SetTimer/KillTimer/WM_TIMER/`_RefreshStatusFromServer` | 全部回退 |
| WeaselServer IPC 改动 | `SetOption(0, "ascii_mode", ...)` 一律 broadcast | 回退到 `global_ascii` gating |
| IPC 开销 | 每 2s/实例一次 ProcessKeyEvent(0) round-trip | 零（只有在 AppME 真正触发时才有一次 Shift keystroke） |
| 刷新延迟 | ≤2 秒 | 立即（一个 keystroke 周期 ~10ms） |
| 架构优雅 | 丑陋：轮询一个本不该轮询的状态 | 干净：复用自然路径 |

### v3 拒绝的备选（与 v1/v2 略有不同）

- **不调 `WeaselServer.exe /ascii`**：v3 不再需要 server 端广播——前台 WeaselTSF 的 OnKeyDown 已经把 server session 状态 toggle 完成。其它进程 session 自然保持各自状态，互不干扰（vim-im-select 用户也是这样切前台）
- **不用 Send `{Shift}` 而用 Send `{LShift}`**：RIME `ascii_composer` 默认配置 `Shift_L: inline_ascii`（lone release toggle）、`Shift_R: commit_code`（差异行为），为保险起见发 Left Shift；若用户改了 RIME 配置使 Shift_L 行为不同则需调整 AHK 脚本
- **保持 `IME_GetMode` 鉴别后台仍调 im-control**：现有 `ime-control -g` 是轻量只读 (`-g get-keyboard`)，不预写 compartment，状态用完成后立即卸载；属可接受开销；如要彻底远离 im-control 可考虑直接读 `GUID_COMPARTMENT_KEYBOARD_INPUTMODE_CONVERSION`，但那又得走 im-control 的注入 hook 即必要性不变

## 四次实施（v3 部署后）：恢复原始轻量 `_ReconcileCompartment` 2s 定时器

v3 部署后实测结果：

| 场景 | Win10 | Win11 |
|------|------|-------|
| AppME 闲置 8s 切换 | ✅ | ✅ |
| gvim 进出 insert/command | ✅ | ❌ 不稳定，时好时坏 |

Win11 gvim 在 commit `830eb55` 时是正常的，到 v3 (`f65d4b8`) 出现回归。

**根因**：`830eb55` 的 2s `SetTimer` 调的是**纯本地 `_ReconcileCompartment`**——只比对 `_status` 与本线程 compartment，零 IPC、零 broadcast。它在 Win11 gvim 偶发 race（im-control 的外部 CONVERSION compartment 写入偶尔没正常触发 OnChange）时起兜底作用：2s 内必然有一次本地比对并刷新 LBB。

commit `8d3f406` 在引入进程内广播时**误删了这个轻量 timer**——本是改造而非替换，结果把它和 v2 后来加的重量 `_RefreshStatusFromServer` 一起认为是"2s 定时器"了。f65d4b8 又只回退 v2，不知这个原始轻量 timer 才是 Win11 gvim 的安全网，导致回归。

### 两种 2s 定时器的关键区分

| 项 | 原 `_ReconcileCompartment` timer (`830eb55`) | `_RefreshStatusFromServer` timer (v2 / commit 31b914d) |
|----|---------------------------------------------|--------------------------------------------------------|
| 比对内容 | 本地 `_status` vs 本线程 compartment | server 端权威态 |
| 是否 IPC | **否**——纯本地比对 | 是——每次 `ProcessKeyEvent(0)` 一轮 IPC |
| 触发 `_UpdateLanguageBar` | 仅在 mismatch 时 | 总是 |
| 架构意义 | 防漏检 OnChange 的安全网 | 跨进程拉权威态的主路径 |
| Ugly? | 不丑，常驻不留痕 | 丑——空载也持续 burn IPC |

用户之前说"2s 定时器 ugly"，是指后者——v2 的 server 轮询。原 `_ReconcileCompartment` 完全不同,它本身在 830eb55 就存在,而且是 Win11 gvim 正常工作的必要条件。

### v4 改动

`WeaselTSF/WeaselTSF.cpp`：
- `_InitDeferredWindow`：恢复 `SetTimer(hWnd, 1, 2000, NULL)`
- `_UninitDeferredWindow`：恢复 `KillTimer(_hDeferredMsgWnd, 1)`
- `_DeferredWndProc`：恢复 `WM_TIMER && wParam == 1` 分支调 `pThis->_ReconcileCompartment()`

这是 commit `830eb55` 原始 `_DeferredWndProc` 的精确还原。v1 的进程内广播(`WM_APP+101` + `_OnRemoteAsciiChange` + `Weasel_*Instance`)保留作 defense-in-depth。`_RefreshStatusFromServer` 不再回来。

### v3 AppME Shift 与 v4 timer 正交互补

| 场景 | 谁负责 |
|------|---------|
| AppME 闲置 8s（无 keystroke、无 focus） | v3 AppME Send Shift（前台 OnKeyDown 主路径）× 同进程多线程兜底（v1 广播如适用） |
| gvim i/Esc（vim-im-select 写外部 compartment） | CONVERSION OnChange 主路径 × 2s `_ReconcileCompartment` timer（v4）防漏检 |
| 任何模式下 race 导致 OnChange 漏 | 2s timer 兜底 |

两者作用域不重叠——timer 解决同进程漏检，Shift 解决跨进程闲置。互不破坏（都幂等）。

### v4 验证计划

1. 编译：`build.bat weasel release`
2. 部署：覆盖 system32 / SysWOW64 的 weasel.dll；**WeaselServer.exe 无需重新部署**（v4 未改 server 端）
3. 重启所有 RIME 宿主进程
4. 场景复现：
   - AppME 闲置 8s（v3 已修，v4 应继续 OK）
   - gvim Win10 i/Esc 模式切换（v3 已 OK，v4 应继续 OK）
   - gvim Win11 i/Esc 模式切换（v3 报时好时坏，v4 应稳定 OK——2s 内必刷一次）
5. 主功能回归：手动 Shift、Ctrl+Space、Windows Terminal 焦点切换均正常

## 五次实施（v4 部署后）：Win11 gvim 仍然时好时坏——v5 回退 8d3f406 进程内广播

v4 (`e94cc65`) 部署 + 精简诊断日志（`5b04a27`）后，用户在 Win11 gvim 测试，多次进出 insert 模式仍出现"时好时坏"故障。

### 日志关键观察（`docs/debugview/VIMELNUC04.log`）

gvim 进程（PID 3680）每次模式切换的固定 pattern：

```
t=X.X  WTSF_OPENCLOSE blind-toggle: new ascii=0 LBB=000000000A0D4CB0
t=X.Y  WTSF_Broadcast: ascii=1 self=20820
```

- 每次都进入 OPENCLOSE else 分支（`_isToOpenClose=false`，Win11），盲 toggle 后状态被翻成中文
- 几十毫秒后 im-control 写入的 ~NATIVE 触发 CONVERSION OnChange，进入 mismatch 分支，broadcast 正确触发 ascii=1
- **从未出现 `WTSF_UpdateWeaselStatus: SKIP sink=NULL`** —— sink 缺失假设排除
- LBB 指针稳定 `0xA0D4CB0`，LBB 实例未被销毁重建

### 830eb55 vs e94cc65 的字符级对照

`git diff 830eb55 HEAD -- WeaselTSF/Compartment.cpp WeaselTSF/LanguageBar.cpp WeaselTSF/WeaselTSF.cpp | ...`

唯一差异就在 `8d3f406` 引入的进程内广播机制：

| 差异 | 来源 commit | 是否影响 gvim 单进程场景 |
|------|------------|------------------------|
| `Weasel_RegisterInstance/Unregister/SnapshotInstances` 等注册表 | `8d3f406` | 增加每次按键的 `g_cs` 锁竞争 |
| `ActivateEx/Deactivate` 中调 Register/Unregister | `8d3f406` | 影响 IME activate 时机 |
| `WM_APP+101` 路由 + `_OnRemoteAsciiChange` | `8d3f406` | 单进程 gvim 同进程无他实例，广播 PostMessage 实际无效但占用代码路径 |
| CONVERSION mismatch 分支末尾广播 block | `8d3f406` | 进入 critical section + vector 复制 + 循环（即使无他实例） |

OPENCLOSE handler、CONVERSION mismatch 主路径、2s `_ReconcileCompartment` timer 与 `830eb55` 字符级等价。

### v5 决策：回退 `8d3f406` 进程内广播

将 `8d3f406` 之后所有进程内广播相关工作彻底删除：

| 文件 | 回退内容 |
|------|---------|
| `WeaselTSF/Globals.h` | 删 `<vector>` include、删 `class WeaselTSF;` 前向声明、删 `g_weaselInstances` 与三个注册表函数声明 |
| `WeaselTSF/Globals.cpp` | 删 `<algorithm>` include、删注册表实现 |
| `WeaselTSF/WeaselTSF.h` | 删 `_GetDeferredWnd()` 公有 accessor、删 `_OnRemoteAsciiChange(bool)` 私有方法声明 |
| `WeaselTSF/WeaselTSF.cpp` | 删 `<resource.h>` include；`ActivateEx` 末尾不再 `Weasel_RegisterInstance`；`Deactivate` 开头不再 `Weasel_UnregisterInstance`；`_DeferredWndProc` 删 `WM_APP+101` 分支；删 `_OnRemoteAsciiChange` 实现 |
| `WeaselTSF/Compartment.cpp` | CONVERSION mismatch 分支末尾删广播 block（不再 `Weasel_SnapshotInstances` 取锁 + 循环 PostMessage） |
| `WeaselTSF/LanguageBar.cpp` | 保留 `WTSF_Reconcile MISMATCH`、`WTSF_UpdateWeaselStatus: SKIP sink=NULL` 诊断日志（频次极低，对 DebugView 无压力） |

v5 状态 = `830eb55` + v3 VimReader AppME Shift 路径。`v3 AppME` 在 AppME 闲置场景下前提供跨进程同步。gvim 进出 insert 走 `830eb55` 已知稳定的 OPENCLOSE + CONVERSION 路径。

### v5 验证计划

1. 编译：`build.bat weasel release`
2. 部署：覆盖 system32 / SysWOW64 的 weasel.dll；**WeaselServer.exe 无需重新部署**
3. 重启 gvim.exe（Win11）与其他 RIME 宿主进程
4. 复现场景：
   - gvim Win11 i/Esc 模式切换，重复 10+ 次，每 5 次观察一次"光标 LBB 是否正确显 A"
   - AppME 闲置 8s（v3 VimReader 提供同步，应继续 OK）
5. 主功能回归：手动 Shift、Ctrl+Space、Windows Terminal 焦点切换均正常

### v5 假设与风险

- **假设**：`8d3f406` 的进程内广播在 gvim 单进程内虽无实际同步对象可发，但 `EnterCriticalSection` + `vector` snapshot + 循环本身可能引入一次额外的延迟和锁竞争，对照 `830eb55` 不存在该路径
- **风险**：若 v5 仍不稳定，则根因不在此，问题在原始 OPENCLOSE 盲 toggle 与 im-control CONVERSION 写入的 race；届时需要进一步回退或重新设计 OPENCLOSE else 分支行为
- **不撤 v3 VimReader 修改**：AppME Send Shift 路径与 weasel 端正交

## 六次实施（v5 部署后）：Win11 gvim 仍不稳定 + Win10 gvim 出现 RIME 被禁用回归

v5 (`38a5fcc`) 部署后实测：
- **Win11 gvim** 仍"时好时坏"，没有改善
- **Win10 gvim** 出现 **RIME 被禁用**回归——这是归档文档 `compartment-external-control-fix.md` §5 早已解决的问题，说明 v3-v5 的演进破坏了归档方案的某条保护

### 用户洞察：方案选错方向

用户指出：
1. 归档方案 `compartment-external-control-fix.md` 用"**值驱动 + `_isToOpenClose` 分支**"解决 Win10/Win11 gvim RIME 被禁用问题
2. 但我们 v3 引入的"AppME Send Shift"路径本质上是 **blind toggle**——与归档方案"值驱动"原则**完全对立**
3. v3 的 Shift 方案虽修复了 AppME 闲置场景，但破坏了归档方案的对称性，造成"顾此失彼"

### 真正根因：OPENCLOSE else 分支的盲 toggle + _UpdateLanguageBar 写回

仔细 trace `VIMELNUC04.log` 的每次 gvim Esc/i：

```
t=X.X  WTSF_OPENCLOSE blind-toggle: new ascii=0    ← OPENCLOSE OnChange else 分支盲 toggle 1→0
t=X.Y  WTSF_Broadcast: ascii=1                      ← 之后 im-control ~NATIVE 触发 CONVERSION OnChange mismatch，_status 0→1
```

**两条 OnChange 之间的到达顺序不确定 → race**：

| 时序 | 结果 |
|------|------|
| im-control ~NATIVE **先** 到达 → OPENCLOSE **后** 到达 | im-control 把 `_status=1`、LBB=A。OPENCLOSE else 分支盲 toggle `_status=1→0`，调 `_UpdateLanguageBar(ascii=0)` 写 compartment=NATIVE on **覆盖** im-control 的 ~NATIVE。LBB 刷中文。**没有后续 OnChange 纠回（compartment 值没新变化）→ 死锁中文** ❌ |
| OPENCLOSE **先** 到达 → im-control ~NATIVE **后** 到达 | OPENCLOSE 盲 toggle `_status→0`、写 NATIVE on。im-control 写 ~NATIVE 触发 mismatch，`_status=1`、LBB=A ✓ |

这就是"时好时坏"！完全与归档文档 §1 描述的"撤销外部变更"旧 bug 同型。

### 归档方案为什么没修这个

归档方案 §1-§8 全部改 **CONVERSION handler**，没动 OPENCLOSE else 分支。OPENCLOSE else 分支的盲 toggle + `_UpdateLanguageBar` 写回是历史遗留代码，与"值驱动"原则对立。当时 Win11 gvim 不稳定的 race 在归档方案阶段没暴露，是因为：
- 归档方案 §5 让 CONVERSION handler 在 `_isToOpenClose=true` 时调 `_SetKeyboardOpen(true)` 重开键盘。这避免了 Win10 gvim ESC 关键盘后无人重开。
- 但 OPENCLOSE else 分支（`_isToOpenClose=false` 即 Win11）的盲 toggle + _UpdateLanguageBar 写回未被审查

v5 的 v3 VimReader Send Shift 只是叠加了 AppME 闲置场景的修复，**没改 OPENCLOSE else 分支**——这是 race 始终存在的根源。

### v6 方案：值驱动贯彻到 OPENCLOSE else 分支 + AppME 改用 im-control + F13

用户洞察"**类似 F13 等不存在的按键 + 值驱动 + `_isToOpenClose` 分支**"给出正确方向。两条改动：

**改动 A（WeaselTSF/Compartment.cpp OPENCLOSE else 分支）**——彻底贯彻值驱动：

```cpp
} else {
  _SetKeyboardOpen(true);                                          // 保底重开键盘（归档方案 §5）
  if (_pLangBarButton && _pLangBarButton->IsLangBarDisabled())
    _EnableLanguageBar(true);
  DWORD convFlags;
  if (SUCCEEDED(_GetCompartmentDWORD(convFlags,
                                      GUID_COMPARTMENT_KEYBOARD_INPUTMODE_CONVERSION))) {
    bool desiredAscii = !(convFlags & TF_CONVERSIONMODE_NATIVE);   // 值驱动
    if (desiredAscii != _status.ascii_mode) {
      _status.ascii_mode = desiredAscii;                           // 同步 state，不盲 toggle
      if (_pEditSessionContext)
        m_client.ClearComposition();
      if (_pLangBarButton)
        _pLangBarButton->UpdateWeaselStatus(_status);             // 只刷 LBB
    }
  }
  // **不调 _UpdateLanguageBar，不写 CONVERSION compartment** —— 避免覆盖 im-control 写入
  // **不调 _HandleLangBarMenuSelect** —— RIME session 切换由 CONVERSION OnChange 处理
}
```

关键改动 vs 原代码：
- 删 `_status.ascii_mode = !_status.ascii_mode`（盲 toggle）
- 删 `_HandleLangBarMenuSelect(...)`（让 CONVERSION handler 独自拥有 RIME session 切换权）
- 删 `_UpdateLanguageBar(_status)`（避免写 compartment 覆盖外部）
- 加 `_GetCompartmentDWORD` + `_status` 同步（值驱动）
- 加 `_pLangBarButton->UpdateWeaselStatus` 刷 LBB
- 保留 `_SetKeyboardOpen(true)` 重开键盘（归档方案 §5 保底，防止 gvim 关键盘后 RIME 被禁用）

**改动 B（VimReader IME.ahk + AppIME.ahk）**——AppME 用 im-control + F13 替换 Send Shift：

```ahk
IME_SetEnglishViaF13() {
    IME_SetAlphanumeric()                        ; im-control -c alphanumeric + WeaselServer /ascii
                                                 ; 写 CONVERSION compartment ~NATIVE
                                                 ; server 切前台 session ascii_mode=true（值驱动）
    Sleep(20)
    Send("{F13 down}")                            ; 触发前台 WeaselTSF::OnKeyDown
    Sleep(10)                                     ; → ProcessKeyEvent(F13) → server 不识别
    Send("{F13 up}")                              ;                            但返回 status.ascii_mode=true
                                                 ; → DoEditSession → GetResponseData
                                                 ; → _UpdateLanguageBar → LBB 刷 A
}
```

为什么 F13 不是 Shift：
- Shift 被 RIME `ascii_composer` 解释为 toggle 信号 → RIME server 状态被盲 toggle（导致 directional 不可控）
- F13 不被 RIME 处理 → server 状态保持 im-control 写入的值（值驱动）
- F13 走 keystroke path 触发 `DoEditSession` 拉权威态刷 LBB（同步）

### v6 改动文件清单

| 端 | 文件 | 改动 |
|------|------|------|
| Weasel | `WeaselTSF/Compartment.cpp` | OPENCLOSE else 分支重写为值驱动 |
| AppME | `C:\Apps\VimReader\lib\utils\IME.ahk` | `IME_SetEnglishViaShift` 重命名为 `IME_SetEnglishViaF13`，体改为 im-control + F13 |
| AppME | `C:\Apps\VimReader\lib\system\AppIME.ahk` | 调用点改用 `IME_SetEnglishViaF13`，更新文档说明 |

### v6 时序分析（验证 race 是否根治）

**gvim Esc 时场景**（_isToOpenClose=false，Win11）：

Case A — im-control ~NATIVE **先** 到，OPENCLOSE **后** 到：
1. im-control 写 compartment = ~NATIVE → OnChange(CONVERSION) → _HandleCompartment(CONVERSION)
   - _status.ascii=1 (insert 时状态)
   - convMode = ~NATIVE, desiredAscii=true
   - desiredAscii(1) != _status(1)? **不真** → 进 else 分支（无操作）
   - LBB 保持 A
2. gvim 写 OPENCLOSE=0 → OnChange(OPENCLOSE) → _isToOpenClose=false → else 分支
   - _SetKeyboardOpen(true)（重开键盘）
   - 读 convFlags = ~NATIVE, desiredAscii=true
   - desiredAscii(1) != _status(1)? **不真**（im-control 已让 _status=1）→ 无 mismatch
   - **不写 compartment、不切 RIME** → LBB 保持 A ✓

Case B — OPENCLOSE **先** 到，im-control ~NATIVE **后** 到：
1. gvim 写 OPENCLOSE=0 → OnChange(OPENCLOSE) → else 分支
   - _SetKeyboardOpen(true)
   - 读 convFlags = 当前 compartment 状态（insert 模式时可能是 NATIVE on 中文 或 ~NATIVE 英文，取决于 insert 时状态）
   - 假设 insert 时是英文：convFlags=~NATIVE, desiredAscii=1, _status=1 → 一致 → 无操作
   - 假设 insert 时是中文：convFlags=NATIVE on, desiredAscii=0, _status=0 → 一致 → 无操作
   - **不写 compartment**
2. im-control 写 ~NATIVE → OnChange(CONVERSION)（值变 true）→ mismatch 处理
   - 读 convMode=~NATIVE, desiredAscii=true
   - _status.ascii=1 → 一致（compartment 同步了）→ 无 mismatch 无操作

无论顺序如何，都不能产生死锁 race。✓

**AppME 闲置 8s 触发**：
1. AppME 调 `IME_SetAlphanumeric` → im-control 写 ~NATIVE + WeaselServer /ascii 切 server session
2. AppME `Sleep(20)` 等 im-control 完成
3. AppME `Send {F13}` → 前台 WeaselTSF::OnKeyDown → ProcessKeyEvent → server 不响应但回 status.ascii=true → DoEditSession → _status.ascii=true → _UpdateLanguageBar → LBB 刷 A ✓

### v6 安全性原则

- **Win10 gvim RIME 被禁用不再回归**：OPENCLOSE else 分支保留 `_SetKeyboardOpen(true)` 这条保底（归档方案 §5）
- **值驱动贯彻到 OPENCLOSE else 分支**：不再盲 toggle _status；不再 _UpdateLanguageBar 写回 compartment；不再 _HandleLangBarMenuSelect 切 RIME。RIME session 切换由 CONVERSION handler 单独负责
- **AppME 不引入 blind toggle**：F13 不被 RIME 处理，server session 状态只受 im-control 操作
- **Ctrl+Space 主功能不受影响**：Ctrl+Space 触发 OPENCLOSE OnChange，else 分支读 CONVERSION 同步 _status，**不切 RIME ascii_mode**——这正是用户期待（Ctrl+Space 切 IME 开关，不应改 RIME ascii_mode）
- **手动 Shift 主功能不受影响**：Shift 触发 OnKeyDown → RIME ascii_composer server 端 toggle → DoEditSession 拉 status → _UpdateLanguageBar 写 compartment → OnChange(CONVERSION) → _HandleCompartment(CONVERSION) 走值驱动分支同步 _status,刷 LBB

### v6 验证计划

1. 编译：`build.bat weasel release`
2. 部署：覆盖 system32 / SysWOW64 的 weasel.dll；WeaselServer.exe 无需重新部署
3. 重启 gvim.exe（Win10/Win11）、Windows Terminal、Total Commander 等所有 RIME 宿主
4. 重新加载 VimReader AHK 脚本（让 `IME_SetEnglishViaF13` 生效）
5. 复现场景：
   - gvim Win11 i/Esc 模式切换，重复 10+ 次，**期望稳定显 A**
   - gvim Win10 i/Esc 模式切换，**期望不出现 RIME 被禁用**
   - AppME 闲置 8s（Win10/Win11），**期望光标 LBB 显 A**
6. 主功能回归：手动 Shift、Ctrl+Space、Windows Terminal 焦点切换均正常

### v3-v6 演进对照

| 项 | v3 (Shift) | v4 (+timer) | v5 (-broadcast) | v6 (F13 + 值驱动 OPENCLOSE) |
|----|------------|-------------|------------------|------------------------------|
| AppME 同步机理 | RIME blind toggle by Shift | + 2s 本地 reconcile 兜底 | 同 v4（清理 v1 broadcast） | im-control -c（值驱动）+ F13 keystroke path 同步 |
| OPENCLOSE else 分支 | 盲 toggle + _UpdateLanguageBar（race 源） | 同 v3 | 同 v3 | **值驱动**——读 compartment 同步 _status，不盲 toggle，不写回 |
| Win11 gvim | 时好时坏 | 仍时好时坏 | 仍时好时坏 | 期望稳定 |
| Win10 gvim RIME 被禁用 | OK | OK | 回归 | 期望 OK（_SetKeyboardOpen 保底） |
| 架构原则 | blind toggle（与归档方案冲突） | 同 | 同 | **值驱动 + _isToOpenClose 分支**（与归档方案一致） |
