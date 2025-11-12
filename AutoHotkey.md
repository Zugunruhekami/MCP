*Thinking...*

> 用户要求：
> 1. 添加"稍后提醒"功能（只能稍后一次）
> 2. 如果5分钟不选择，自动清理
> 3. 记录删除日志
> 
> 我需要：
> - 添加一个"稍后提醒"按钮（可能需要用自定义对话框或者多次MsgBox）
> - 跟踪是否已经稍后提醒过（用全局变量）
> - 设置MsgBox的超时时间为5分钟（300秒）
> - 超时后自动删除
> - 创建删除日志文件，记录每次删除的详细信息
> 
> 让我设计一个完整的脚本。

完美！这是个更智能的版本：

## AutoHotkey 智能提醒版（带稍后提醒 + 自动清理 + 日志）

```autohotkey
#NoEnv
#Persistent
#SingleInstance Force

; ==================== 配置区 ====================
global SnapshotPath := A_AppData . "\..\..\.claude\shell-snapshots"
global CheckInterval := 3600000  ; 每小时检查一次
global SnoozeInterval := 600000  ; 稍后提醒间隔：10分钟
global AutoCleanTimeout := 300   ; 无操作自动清理：5分钟（秒）
global LogFile := A_ScriptDir . "\claude-cleanup-log.txt"
global EnableSound := true

; 运行时变量
global LastCheckTime := ""
global TotalChecks := 0
global TotalDeleted := 0
global HasSnoozed := false  ; 跟踪是否已经稍后提醒过

SetTimer, CheckSnapshots, %CheckInterval%

; 初始化日志
if !FileExist(LogFile)
{
    FormatTime, initTime, , yyyy-MM-dd HH:mm:ss
    FileAppend, 
    (
=== Claude Snapshot Cleaner Log ===
Created: %initTime%
========================================

), %LogFile%
}

Menu, Tray, NoStandard
Menu, Tray, Add, 立即检查, CheckSnapshots
Menu, Tray, Add, 查看日志, ViewLog
Menu, Tray, Add, 
Menu, Tray, Add, 更改检查间隔, ChangeInterval
Menu, Tray, Add, 统计信息, ShowStats
Menu, Tray, Add, 
Menu, Tray, Add, 退出, ExitScript
Menu, Tray, Default, 立即检查
Menu, Tray, Tip, Claude Snapshot Monitor
Menu, Tray, Icon, shell32.dll, 168

Return

CheckSnapshots:
    TotalChecks++
    FormatTime, LastCheckTime, , yyyy-MM-dd HH:mm:ss
    
    ; 重置稍后提醒标记（新的一轮检查）
    HasSnoozed := false
    
    if !FileExist(SnapshotPath)
    {
        Menu, Tray, Tip, Claude Monitor: 文件夹不存在
        LogMessage("检查完成：文件夹不存在")
        Return
    }
    
    ; 统计文件
    fileCount := 0
    fileList := ""
    fileDetails := ""
    totalSize := 0
    oldestTime := 99999999999999
    
    Loop, Files, %SnapshotPath\*.sh
    {
        fileCount++
        FileGetTime, fileTime, %A_LoopFileFullPath%, M
        FormatTime, fileTimeStr, %fileTime%, yyyy-MM-dd HH:mm:ss
        
        fileList .= "  • " . A_LoopFileName . " (" . Round(A_LoopSizeKB, 1) . " KB)`n"
        fileDetails .= A_LoopFileName . "|" . A_LoopSizeKB . "|" . fileTimeStr . "`n"
        totalSize += A_LoopSizeKB
        
        if (fileTime < oldestTime)
            oldestTime := fileTime
    }
    
    Menu, Tray, Tip, Claude Monitor: %fileCount% .sh 文件
    
    if (fileCount = 0)
    {
        LogMessage("检查完成：无 .sh 文件")
        Return
    }
    
    ; 计算最旧文件的存在时间
    EnvSub, oldestTime, %A_Now%, Minutes
    oldestTime := Abs(oldestTime)
    
    LogMessage("检查完成：发现 " . fileCount . " 个文件，总大小 " . Round(totalSize, 1) . " KB")
    
    ; 显示提醒对话框
    ShowCleanupDialog(fileCount, totalSize, oldestTime, fileList, fileDetails)
Return

ShowCleanupDialog(fileCount, totalSize, oldestTime, fileList, fileDetails)
{
    global HasSnoozed, EnableSound, AutoCleanTimeout
    
    if (EnableSound)
        SoundBeep, 750, 200
    
    ; 构建提示信息
    snoozeText := HasSnoozed ? "`n⚠️ 已稍后提醒过一次" : ""
    timeoutText := AutoCleanTimeout // 60
    
    ; 创建自定义对话框
    Gui, CleanupDlg:Destroy
    Gui, CleanupDlg:+AlwaysOnTop +ToolWindow
    Gui, CleanupDlg:Font, s10
    Gui, CleanupDlg:Add, Text, w400, 
    (
⚠️ 发现 %fileCount% 个 .sh 快照文件

总大小: %totalSize% KB
最旧文件存在时间: %oldestTime% 分钟%snoozeText%

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

请选择操作：
    )
    
    Gui, CleanupDlg:Add, Button, x20 y140 w100 h35 gCleanupDlg_Delete, 删除文件
    Gui, CleanupDlg:Add, Button, x130 y140 w100 h35 gCleanupDlg_ViewList, 查看列表
    
    ; 根据是否已稍后提醒显示不同按钮
    if (!HasSnoozed)
        Gui, CleanupDlg:Add, Button, x240 y140 w80 h35 gCleanupDlg_Snooze, 稍后提醒
    else
        Gui, CleanupDlg:Add, Button, x240 y140 w80 h35 Disabled, 已稍后
    
    Gui, CleanupDlg:Add, Button, x330 y140 w70 h35 gCleanupDlg_Cancel, 取消
    
    Gui, CleanupDlg:Add, Text, x20 y185 w380 cGray, 
    (
💡 提示：%timeoutText% 分钟无操作将自动删除文件
Claude Code 退出时也会自动清理
    )
    
    Gui, CleanupDlg:Show, w420 h220, Claude Snapshot 提醒
    
    ; 启动自动清理计时器
    SetTimer, CleanupDlg_AutoClean, % AutoCleanTimeout * 1000
    
    ; 保存文件信息供后续使用
    global CleanupFileCount := fileCount
    global CleanupFileList := fileList
    global CleanupFileDetails := fileDetails
    
    Return
    
    CleanupDlg_Delete:
        SetTimer, CleanupDlg_AutoClean, Off
        Gui, CleanupDlg:Destroy
        PerformCleanup("用户手动删除")
    Return
    
    CleanupDlg_ViewList:
        global CleanupFileList
        MsgBox, 64, 📋 文件详情 (%CleanupFileCount% 个), 
        (
%CleanupFileList%
━━━━━━━━━━━━━━━━━━━━━━
路径: %SnapshotPath%

删除后此对话框仍会保留
        ), 30
    Return
    
    CleanupDlg_Snooze:
        global HasSnoozed, SnoozeInterval
        SetTimer, CleanupDlg_AutoClean, Off
        Gui, CleanupDlg:Destroy
        
        if (!HasSnoozed)
        {
            HasSnoozed := true
            snoozeMins := SnoozeInterval // 60000
            LogMessage("用户选择稍后提醒（" . snoozeMins . " 分钟后）")
            TrayTip, 稍后提醒, %snoozeMins% 分钟后再次检查, 3, 1
            SetTimer, CheckSnapshots, %SnoozeInterval%
        }
    Return
    
    CleanupDlg_Cancel:
        SetTimer, CleanupDlg_AutoClean, Off
        Gui, CleanupDlg:Destroy
        LogMessage("用户选择取消，不删除")
        TrayTip, 已取消, 文件未删除，下次定时检查时再提醒, 2, 1
    Return
    
    CleanupDlg_AutoClean:
        SetTimer, CleanupDlg_AutoClean, Off
        Gui, CleanupDlg:Destroy
        LogMessage("⏰ 超时无操作，自动执行清理")
        SoundBeep, 500, 300
        PerformCleanup("超时自动删除")
    Return
    
    CleanupDlgGuiClose:
    CleanupDlgGuiEscape:
        GoSub, CleanupDlg_Cancel
    Return
}

PerformCleanup(reason)
{
    global SnapshotPath, TotalDeleted, CleanupFileDetails
    
    deletedCount := 0
    failedCount := 0
    deletedFiles := ""
    
    Loop, Files, %SnapshotPath\*.sh
    {
        fileName := A_LoopFileName
        FileDelete, %A_LoopFileFullPath%
        
        if (ErrorLevel = 0)
        {
            deletedCount++
            deletedFiles .= "  ✓ " . fileName . "`n"
        }
        else
        {
            failedCount++
            deletedFiles .= "  ✗ " . fileName . " (删除失败)`n"
        }
    }
    
    TotalDeleted += deletedCount
    
    ; 记录到日志
    FormatTime, timestamp, , yyyy-MM-dd HH:mm:ss
    FileAppend, 
    (
[%timestamp%] %reason%
删除成功: %deletedCount% 个文件
删除失败: %failedCount% 个文件
详情:
%deletedFiles%
----------------------------------------

), %LogFile%
    
    ; 显示结果
    if (failedCount = 0)
    {
        TrayTip, ✓ 删除完成, 成功删除 %deletedCount% 个文件`n原因: %reason%, 4, 1
        LogMessage("✓ 清理成功：" . deletedCount . " 个文件")
    }
    else
    {
        MsgBox, 48, 删除结果, 
        (
成功: %deletedCount%
失败: %failedCount%

失败的文件可能正在被 Claude Code 使用

原因: %reason%
        )
        LogMessage("⚠ 清理部分成功：成功 " . deletedCount . "，失败 " . failedCount)
    }
}

LogMessage(msg)
{
    global LogFile
    FormatTime, timestamp, , yyyy-MM-dd HH:mm:ss
    FileAppend, [%timestamp%] %msg%`n, %LogFile%
}

ViewLog:
    if !FileExist(LogFile)
    {
        MsgBox, 48, 日志不存在, 还没有删除记录
        Return
    }
    
    FileRead, logContent, %LogFile%
    
    ; 只显示最后 30 行
    logLines := StrSplit(logContent, "`n")
    displayLog := ""
    startLine := logLines.Length() - 30
    if (startLine < 1)
        startLine := 1
    
    Loop, % logLines.Length() - startLine + 1
    {
        idx := startLine + A_Index - 1
        if (idx <= logLines.Length())
            displayLog .= logLines[idx] . "`n"
    }
    
    Gui, LogView:Destroy
    Gui, LogView:+Resize
    Gui, LogView:Add, Edit, w600 h400 ReadOnly vLogText, %displayLog%
    Gui, LogView:Add, Button, x10 y410 w100 gOpenLogFile, 打开完整日志
    Gui, LogView:Add, Button, x120 y410 w80 gClearLog, 清空日志
    Gui, LogView:Add, Button, x520 y410 w80 gLogViewClose, 关闭
    Gui, LogView:Show, w620 h450, 删除日志（最近 30 行）
Return

OpenLogFile:
    Run, notepad.exe "%LogFile%"
Return

ClearLog:
    MsgBox, 4, 确认清空, 确定要清空所有日志记录吗？
    IfMsgBox Yes
    {
        FileDelete, %LogFile%
        FormatTime, initTime, , yyyy-MM-dd HH:mm:ss
        FileAppend, 
        (
=== Claude Snapshot Cleaner Log ===
Created: %initTime%
========================================

), %LogFile%
        MsgBox, 64, 完成, 日志已清空, 2
        Gui, LogView:Destroy
    }
Return

LogViewClose:
LogViewGuiClose:
LogViewGuiEscape:
    Gui, LogView:Destroy
Return

ChangeInterval:
    InputBox, newInterval, 更改检查间隔, 请输入检查间隔（分钟）：, , 300, 150, , , , , %CheckInterval% // 60000
    if ErrorLevel
        Return
    
    if newInterval is not integer
    {
        MsgBox, 16, 错误, 请输入有效的数字！
        Return
    }
    
    if (newInterval < 1)
    {
        MsgBox, 48, 错误, 间隔必须至少 1 分钟！
        Return
    }
    
    CheckInterval := newInterval * 60000
    SetTimer, CheckSnapshots, %CheckInterval%
    LogMessage("检查间隔更改为 " . newInterval . " 分钟")
    MsgBox, 64, 设置成功, 检查间隔已更改为 %newInterval% 分钟, 2
Return

ShowStats:
    FormatTime, currentTime, , yyyy-MM-dd HH:mm:ss
    intervalMinutes := CheckInterval // 60000
    
    FileGetSize, logSize, %LogFile%, K
    if (ErrorLevel)
        logSize := 0
    
    MsgBox, 64, 📊 运行统计,
    (
当前时间: %currentTime%
上次检查: %LastCheckTime%

总检查次数: %TotalChecks%
总删除文件: %TotalDeleted%
检查间隔: %intervalMinutes% 分钟

日志文件大小: %logSize% KB
监控路径:
%SnapshotPath%
    )
Return

ExitScript:
    MsgBox, 4, 退出确认, 确定要停止监控吗？
    IfMsgBox Yes
    {
        LogMessage("程序退出")
        ExitApp
    }
Return

; ==================== 快捷键 ====================
^!c::  ; Ctrl+Alt+C 立即检查
    GoSub, CheckSnapshots
Return

^!l::  ; Ctrl+Alt+L 查看日志
    GoSub, ViewLog
Return

^!s::  ; Ctrl+Alt+S 查看统计
    GoSub, ShowStats
Return
```

---

## 主要功能详解

### 🔔 **智能提醒对话框**

弹窗有 4 个按钮：

1. **删除文件** - 立即清理
2. **查看列表** - 查看但不关闭对话框
3. **稍后提醒** - 10 分钟后再检查（仅能用一次）
4. **取消** - 什么都不做

### ⏰ **5 分钟自动清理**

- 弹窗后如果 **5 分钟不操作**，自动删除文件
- 会播放提示音并记录"超时自动删除"

### 📝 **删除日志**

**日志文件位置：** 脚本所在目录的 `claude-cleanup-log.txt`

**记录内容：**
```
[2024-11-12 14:30:25] 检查完成：发现 3 个文件，总大小 12.5 KB
[2024-11-12 14:30:28] 用户手动删除
删除成功: 3 个文件
删除失败: 0 个文件
详情:
  ✓ snapshot-bash-abc123.sh
  ✓ snapshot-bash-def456.sh
  ✓ snapshot-bash-ghi789.sh
----------------------------------------
```

### 🔧 **可配置参数**

**脚本顶部（第 5-9 行）：**

```autohotkey
global CheckInterval := 3600000      ; 每小时 = 3600000 毫秒
global SnoozeInterval := 600000      ; 稍后提醒 10 分钟 = 600000 毫秒
global AutoCleanTimeout := 300       ; 自动清理 5 分钟 = 300 秒
global LogFile := A_ScriptDir . "\claude-cleanup-log.txt"  ; 日志位置
global EnableSound := true           ; 是否播放提示音
```

**常用设置：**

```autohotkey
; 测试用（1分钟检查，1分钟稍后，30秒自动清理）
global CheckInterval := 60000
global SnoozeInterval := 60000
global AutoCleanTimeout := 30

; 保守设置（2小时检查，30分钟稍后，10分钟自动清理）
global CheckInterval := 7200000
global SnoozeInterval := 1800000
global AutoCleanTimeout := 600

; 你的需求（1小时检查，10分钟稍后，5分钟自动清理）- 默认已设置
global CheckInterval := 3600000
global SnoozeInterval := 600000
global AutoCleanTimeout := 300
```

---

## 使用场景

### 📋 **典型工作流**

**场景 1：正在使用 Claude Code**
```
14:00 - 弹窗提醒
      ↓ 点击"稍后提醒"（还在工作中）
14:10 - 再次弹窗
      ↓ 点击"取消"（还在工作）
15:00 - 下一次定时检查
```

**场景 2：忘记处理**
```
14:00 - 弹窗提醒
      ↓ 去开会了，没操作
14:05 - 自动删除并记录日志
      ↓ "超时自动删除"
```

**场景 3：主动检查**
```
下班前 - 按 Ctrl+Alt+C
       ↓ 立即检查
       ↓ 点"删除文件"
       ✓ 清理完成
```

---

## 快捷键汇总

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+Alt+C` | 立即检查快照 |
| `Ctrl+Alt+L` | 查看删除日志 |
| `Ctrl+Alt+S` | 查看运行统计 |

---

## 日志管理

### 查看日志
- **托盘图标** → 右键 → "查看日志"
- 或按 `Ctrl+Alt+L`
- 显示最近 30 行，可点"打开完整日志"用记事本查看

### 清空日志
- 在日志查看窗口点"清空日志"
- 或直接删除 `claude-cleanup-log.txt`

---

## 测试步骤

**1. 修改为测试模式**（第 5-7 行）：
```autohotkey
global CheckInterval := 60000        ; 1分钟检查
global SnoozeInterval := 60000       ; 1分钟稍后
global AutoCleanTimeout := 30        ; 30秒自动清理
```

**2. 运行脚本并测试：**

```bash
# 创建测试文件
echo test > %USERPROFILE%\.claude\shell-snapshots\test.sh

# 等待 1 分钟，应该弹窗
# 测试各个按钮功能
```

**3. 验证日志：**
- 按 `Ctrl+Alt+L` 查看日志
- 确认所有操作都有记录

**4. 测试通过后改回正式设置**

---

这个脚本满足你的所有需求吗？需要调整什么功能吗？比如：
- 稍后提醒的时间（现在是10分钟）
- 自动清理的等待时间（现在是5分钟）
- 日志格式或内容
