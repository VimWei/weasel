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
