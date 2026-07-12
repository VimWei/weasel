# 光标位置 IME 状态图标调试记录

唯一问题：**语言栏按钮（`_pLangBarButton`）**——光标位置输入法状态图标不更新。

RIME 托盘图标和任务栏"中"/"A" 指示器均始终正确，从未出问题。

## 三个层级

| 层级 | 位置 | 数据源 | 更新方式 | 状态 |
|------|------|--------|---------|------|
| **RIME 引擎** | 右下角通知区域托盘图标 | RIME 内部 ascii_mode | WeaselServer IPC → 更新托盘图标 | ✅ 始终正确 |
| **任务栏"中"/"A" 指示器** | 任务栏右侧系统托盘区 | `_status.ascii_mode` | `_SetCompartmentDWORD()` 写 TSF compartment → TSF 渲染 | ✅ 始终正确 |
| **语言栏按钮 (`_pLangBarButton`)** | 光标附近的系统语言栏按钮 | `_status.ascii_mode`（同上） | `_pLangBarButton->UpdateWeaselStatus()` → TSF OnUpdate | ❌ 唯一问题 |

## 光标语言栏按钮的更新机制

`_pLangBarButton->UpdateWeaselStatus(stat)` → `_pLangBarItemSink->OnUpdate(TF_LBI_STATUS \| TF_LBI_ICON)` → TSF 刷新当前文本上下文的语言栏按钮。

### 唯二调用点

1. `_UpdateLanguageBar` 末端（`LanguageBar.cpp:430-431`）
2. `_HandleCompartment(CONVERSION)` 内部（`Compartment.cpp:304-305`）

### `_UpdateLanguageBar` 的调用路径

- `OnSetThreadFocus`（`WeaselTSF.cpp:189`）— 焦点切换
- `_HandleCompartment(OPENCLOSE)`（`Compartment.cpp:270,281`）
- `_HandleCompartment(CONVERSION)` → `RequestEditSession` → `DoEditSession`（`Compartment.cpp:316`）
- `_DeferredWndProc(WM_APP+100)`（`WeaselTSF.cpp:242`）

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

说明 `UpdateWeaselStatus` 只刷新**当前活跃文本上下文**的语言栏按钮，旧位置（已失焦）的上下文不接收更新。这也意味着如果 OnChange 跨线程触发且 `UpdateWeaselStatus` 被调用，它只影响调用时刻的活跃上下文。

## 待验证（通过 DebugView 日志）

1. **OnChange(CONVERSION) 是否跨线程触发？** — im-control 在 hook 线程写 compartment → TSF 是否在 WeaselTSF 线程触发 `_HandleCompartment(CONVERSION)`？（触发则执行内部 `UpdateWeaselStatus`，不触发则永不知道 compartment 变了）
2. **`_pLangBarButton` 在 AppIME 进程是否为 NULL？** — Total Commander / Windows Terminal 中 `_pLangBarButton` 是否存在？（NULL 则 `UpdateWeaselStatus` 静默跳过）

## Debug 日志已添加

| 位置 | 前缀 | 用途 |
|------|------|------|
| `_HandleCompartment(OPENCLOSE)` 入口 | `WTSF_OnChange(OPENCLOSE)` | 确认 open/close 触发 |
| `_HandleCompartment(CONVERSION)` 入口 | `WTSF_OnChange(CONVERSION)` | 确认 conversion 跨线程触发 |
| `_ReconcileCompartment` | `WTSF_Reconcile` | 检查 `_pLangBarButton` 状态和 mismatch |

## 编译部署

```powershell
cd C:\Apps\git-kb\repos\VimWei\weasel
build.bat weasel release
Rename-Item C:\Windows\system32\weasel.dll weasel.dll.bak -Force
Copy-Item output\weaselx64.dll C:\Windows\system32\weasel.dll -Force
Rename-Item C:\Windows\SysWOW64\weasel.dll weasel.dll.bak -Force
Copy-Item output\weasel.dll C:\Windows\SysWOW64\weasel.dll -Force
```

重启目标应用，运行 DebugView。
