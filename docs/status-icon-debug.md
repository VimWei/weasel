# WeaselTSF 状态图标调试记录

## 问题描述

使用 im-control 控制 RIME/weasel 输入法的中英文状态时，出现以下问题：

1. **VimReader + Windows Terminal**：手动切换到中文状态，等待8秒后 AppIME 自动切换到英文，右下角托盘图标显示正确（"A"），但光标位置的 WeaselPanel 图标显示"中"

2. **gvim**：使用 i 和 esc 切换模式，win 11 光标位置的图标显示不正确，而win 10 图标是正确的。

## 相关仓库

- `c:\Apps\git-kb\repos\VimWei\im-control\` - 输入法控制工具
- `c:\Apps\git-kb\repos\VimWei\weasel\` - 小狼毫输入法
- `c:\Apps\VimReader\lib\system\AppIME.ahk` - AppIME 自动切换
- `c:\Apps\vim-init\pack\mydev\opt\vim-im-select\` - vim 输入法切换插件

## 调查发现

### 1. WeaselServer 状态正确

右下角托盘图标显示正确（"A"），说明 WeaselServer 的 RIME 引擎状态是正确的。

### 2. WeaselTSF 状态更新链完整

通过 DebugView 调试输出，确认整个更新链都正确执行：

```
[1] _HandleCompartment: desiredAsciiMode=true
[1] _HandleCompartment: _status.ascii_mode=false
[2] _HandleCompartment: _status.ascii_mode updated
[3] _HandleCompartment: calling _HandleLangBarMenuSelect
[4] _HandleCompartment: calling _UpdateLanguageBar
[5] _HandleCompartment: calling _cand->UpdateUI
CCandidateList::UpdateUI: ascii_mode=true
UI::Update: ascii_mode=true
UIImpl::Refresh: called
WeaselPanel::Refresh: m_status.ascii_mode=true
[7] WeaselPanel::Refresh: ctx_changed=false
[8] WeaselPanel::Refresh: status_changed=true
[9] WeaselPanel::Refresh: calling RedrawWindow
[11] WeaselPanel::DrawIcon: ShouldDisplayStatusIcon=true
[12] WeaselPanel::DrawIcon: m_status.ascii_mode=true -> m_iconAlpha
[10] WeaselPanel::Refresh: RedrawWindow completed
[6] _HandleCompartment: _cand->UpdateUI completed
```

### 3. 图标句柄不同

调试输出显示 `m_iconAlpha` 和 `m_iconEnabled` 有不同的句柄值：
```
[12] DrawIcon: m_iconAlpha=1578370695, m_iconEnabled=464127479
```

### 4. 图标文件正确

- `en.ico` 文件大小：46850 字节（用户确认显示"A"图标）
- `zh.ico` 文件大小：38611 字节（显示"中"图标）
- WeaselTSF.rc 正确配置：`IDI_EN ICON "..\\resource\\en.ico"`

## 已实施的修复

### 修复1：Compartment.cpp - 添加 WeaselTSF UI 更新

在 `_HandleCompartment` 中，当 `GUID_COMPARTMENT_KEYBOARD_INPUTMODE_CONVERSION` 变更时，调用 `_cand->UpdateUI()` 更新 WeaselTSF 的 UI：

```cpp
_HandleLangBarMenuSelect(_status.ascii_mode
                             ? ID_WEASELTRAY_ENABLE_ASCII
                             : ID_WEASELTRAY_DISABLE_ASCII);
if (_pEditSessionContext)
  m_client.ClearComposition();
_UpdateLanguageBar(_status);
// 更新 WeaselTSF 的 UI 状态（光标位置图标）
_cand->UpdateUI(weasel::Context(), _status);
```

### 修复2：WeaselPanel.cpp - 添加状态变化检测

在 `WeaselPanel::Refresh()` 中，同时检查 `m_ctx` 和 `m_status` 的变化：

```cpp
if (m_ctx != m_octx || m_status != m_ostatus) {
  m_octx = m_ctx;
  m_ostatus = m_status;
  RedrawWindow();
}
```

### 修复3：WeaselIPCData.h - 添加 operator!=

在 `Status` 结构体中添加 `operator!=`：

```cpp
bool operator!=(const Status status) const { return !(*this == status); }
```

### 修复4：CandidateList.cpp - 确保 UI 面板创建

在 `CCandidateList::UpdateUI` 中，先更新状态再创建面板：

```cpp
// 先更新状态
_ui->Update(ctx, status);
// 再创建面板（确保 m_status 引用的是最新状态）
_MakeUIWindow();
// 刷新面板
_ui->Refresh();
```

## 遗留问题

### 问题：图标显示仍然不正确

尽管调试输出显示所有步骤都正确执行：
- `m_status.ascii_mode=true`
- 选择了 `m_iconAlpha`
- `RedrawWindow()` 被调用

但用户看到的仍然是"中"图标。

### 可能的原因

1. **图标资源加载问题**：`LoadIconW(IDI_EN, ...)` 可能加载了错误的图标资源
2. **多个 UI 实例**：可能存在多个 WeaselPanel 实例，其中一个显示旧图标
3. **Windows TSF 框架状态指示器**：用户看到的可能是 Windows 系统自带的输入法状态指示器，而不是 WeaselPanel
4. **图标缓存**：Windows 可能缓存了旧的图标

### 下一步调查方向

1. 确认用户看到的"中"图标是否真的来自 WeaselPanel
2. 检查是否有其他 UI 组件在显示状态图标
3. 检查 Windows TSF 框架的状态指示器机制

## 编译部署

```powershell
cd C:\Apps\git-kb\repos\VimWei\weasel
build.bat weasel release

# 部署（管理员权限）
Rename-Item C:\Windows\system32\weasel.dll weasel.dll.bak -Force
Copy-Item output\weaselx64.dll C:\Windows\system32\weasel.dll -Force
Rename-Item C:\Windows\SysWOW64\weasel.dll weasel.dll.bak -Force
Copy-Item output\weasel.dll C:\Windows\SysWOW64\weasel.dll -Force

# 重启使用 RIME 的应用
```

## 调试方法

使用 DebugView 查看调试输出：
1. 运行 `dbgview64.exe`
2. 菜单 → Capture → Capture Win32（确保已勾选）
3. 在目标应用中触发 im-control 切换
4. 观察调试输出
