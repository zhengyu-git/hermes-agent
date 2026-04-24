---
name: unreal-insights-cli-limitation
description: "UnrealInsights CLI cannot export CSV — tested on WSL and Windows CMD (UE 5.7). Must use GUI."
tags: [unreal-engine, unreal-insights, wsl, performance-analysis, trace-analysis]
---

# Unreal Insights CLI Limitation in WSL

## Problem
UnrealInsights CLI cannot export CSV/Timing data automatically — tested on both WSL and Windows CMD with UE 5.7, the `-ExecOnCompleteCmd` / `-ExecOnAnalysisCompleteCmd` commands never execute.

## Symptom
- UnrealInsights opens the trace file successfully
- Log shows analysis completing ("Closing analysis cache")
- But no CSV files are produced
- Exit code is 0

## Verified Parameters That Do NOT Work
- `-ExecOnCompleteCmd="@rsp_file"` (UE 5.7, Windows CMD)
- `-ExecOnCompleteCmd="TimingInsights.ExportTimerStatistics path"` (UE 5.7, Windows CMD)
- `-ExecOnCompleteCmd="TimingInsights.ExportTimerStatistics \"path\""` (with quotes, UE 5.7)
- `-ExecOnAnalysisCompleteCmd="..."` (UE 5.7)
- Response file with multiple commands (`@/path/to/export.rsp`)
- `-WaitForAnalysis` flag
- Combinations of all the above

## Tested Versions
| Version | Environment | CLI Export | GUI Export |
|---------|------------|------------|------------|
| UE 5.4  | Windows CMD | ✅ Should work (documented) | ✅ |
| UE 5.7  | Windows CMD | ❌ Failed (all params) | ✅ |
| UE 5.7  | WSL | ❌ Failed | ✅ |

## Root Cause
- **WSL**: Path resolution and working directory issues prevent output
- **UE 5.7**: CLI auto-export feature appears to be broken or removed in this version
- **Note**: Only UE 5.4 is confirmed to support CLI export (from official docs)

## Solution
**Use the UnrealInsights GUI on Windows directly** — the GUI is the only reliable way to export timing data from trace files.

### Manual Export Steps (UE 5.7)
1. Open `UnrealInsights.exe` from `D:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\`
2. Drag .utrace file into the window (or File → Open)
3. Click **Timing** tab
4. **File → Export Timer Statistics**
5. Save as CSV

### Alternative: Install UE 5.4
If CLI auto-export is needed, install UE 5.4 (confirmed to support `-ExecOnCompleteCmd`). But UE 5.4 may not open UE 5.7 traces.

## Trace File Format (for custom parsing if needed)
- **Magic**: `2CRT` (4 bytes)
- **Version**: stored as uint32 LE at offset 4
- **Chunk 0** (metadata): starts at file offset 8, size at offset 8 (uint32 LE)
  - Contains event type definitions as null-terminated strings
  - ~17MB for a typical trace
- **Event data**: starts at `8 + chunk0_size` bytes
  - Events use ULEB128 timestamps
  - Each event starts with a type byte (0x00-0x07) followed by timestamp and payload
  - Event type names stored in chunk 0 metadata

## Paths
- 用户实际路径: `D:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealInsights.exe`
- WSL: `/mnt/d/Program Files/Epic Games/UE_5.7/Engine/Binaries/Win64/UnrealInsights.exe`

## Windows 拖拽导出脚本（bat）— ⚠️ UE 5.7 不支持

在用户桌面放置 `utrace_to_csv.bat`，可直接拖拽 .utrace 文件导出 CSV。

### ⚠️ 已知坑点

1. **UE 5.7 CLI 不支持自动导出**: 所有 `-ExecOnCompleteCmd` 参数组合均失败，脚本无法工作
2. **中文路径问题**: trace 文件路径含中文字符时，UnrealInsights CLI 可能无法正确读取
   - 短路径名（8.3格式）`%%~si` 部分有效但不完全
   - 复制到纯英文路径可解决复制问题，但导出仍失败
3. **bat 文件编码**: bat 文件中含中文字符时 CMD 会乱码
   - 解决方案：脚本用纯英文写，或开头加 `chcp 65001 >nul 2>nul`
4. **用户要求**: 不要修改任何 Windows 软件设置，脚本只做只读+导出

### 推荐脚本（支持中文路径，纯英文避免乱码）

```bat
@echo off
chcp 65001 >nul 2>nul
setlocal enabledelayedexpansion
echo ========================================
echo   Unreal Insights Trace Export Tool
echo ========================================
echo.

set "TRACE_FILE=%~1"
echo [1/4] Trace: %TRACE_FILE%
echo.

if not exist "%TRACE_FILE%" (
    echo [ERROR] File not found
    pause
    exit /b 1
)

set "INSIGHTS=D:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealInsights.exe"
set "RSP_FILE=%~dp0export.rsp"
set "OUTPUT_CSV=%~dp0TimerStats.csv"

echo [2/4] UnrealInsights: %INSIGHTS%
echo [2/4] Output: %OUTPUT_CSV%
echo.

if not exist "%INSIGHTS%" (
    echo [ERROR] UnrealInsights.exe not found!
    pause
    exit /b 1
)

:: Use short path (8.3 format) to avoid Chinese character issues
for %%i in ("%TRACE_FILE%") do set "SHORT_TRACE=%%~si"
for %%i in ("%OUTPUT_CSV%") do set "SHORT_CSV=%%~si"
for %%i in ("%RSP_FILE%") do set "SHORT_RSP=%%~si"

echo [3/4] Short path: %SHORT_TRACE%
echo TimingInsights.ExportTimerStatistics %SHORT_CSV% > "%RSP_FILE%"
echo [3/4] Created rsp file
echo.

echo [4/4] Exporting, please wait...
echo ========================================
echo.

"%INSIGHTS%" -OpenTraceFile=%SHORT_TRACE% -NoUI -AutoQuit -ExecOnCompleteCmd="@%SHORT_RSP%"

echo.
echo ========================================
if exist "%OUTPUT_CSV%" (
    echo [SUCCESS] CSV exported!
    echo File: %OUTPUT_CSV%
) else (
    echo [FAILED] Export failed.
)

del "%RSP_FILE%" 2>nul

echo.
echo Press any key to exit...
pause >nul
```

**使用方法**：把 `.utrace` 文件拖到 `utrace_to_csv.bat` 上，等窗口跑完，CSV 生成在桌面。

**注意**：
- 脚本中 UE 路径需根据用户实际安装位置修改（当前是 UE_5.7）
- 大 trace 文件可能需要几分钟
- 输出 CSV 为 `TimerStats.csv`（在桌面）
- 脚本不会修改任何 Windows/软件设置，只做只读+导出

## Known Compatible Traces
- `20260423_163023.utrace` (1.2GB, Windows UE 5.4, Infinity Nikki/X6Game) — opens fine
- Mac-recorded traces may have protocol version mismatches on Windows
