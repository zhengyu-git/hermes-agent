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

## Verified Parameters That Do NOT Work (UE 5.4 + UE 5.7)
- `-ExecOnCompleteCmd="@rsp_file"` (UE 5.4 & 5.7)
- `-ExecOnCompleteCmd="TimingInsights.ExportTimerStatistics path"` (UE 5.4 & 5.7)
- `-ExecOnAnalysisCompleteCmd="..."` (UE 5.4 & 5.7)
- `-ExecCmds="..."` (UE 5.4)
- `-OpenTraceFile="..."` (UE 5.4 — 不触发分析，直接退出)
- `-trace="..."` (UE 5.4 — 同上)
- `-timing` / `-timing=cpu` (UE 5.4 — 不触发分析)
- 去掉 `-NoUI` 加 `-AutoQuit` (UE 5.4 — 同上)
- `-WaitForAnalysis` flag
- Combinations of all the above

## Tested Versions
| Version | Environment | CLI Export | GUI Export |
|---------|------------|------------|------------|
| UE 5.4  | Windows CMD | ❌ Failed (all params, 2026-04-27 实测) | ✅ |
| UE 5.4  | Windows CMD (中文路径) | ❌ Failed (pushd绕过中文路径后仍不分析) | ✅ |
| UE 5.7  | Windows CMD | ❌ Failed (all params) | ✅ |
| UE 5.7  | WSL | ❌ Failed | ✅ |

## Root Cause
- **UE 5.4**: 虽然文档说支持 `-ExecOnCompleteCmd`，但实测 2026-04-27 发现完全不工作。UnrealInsights 启动后不分析 trace 文件，日志只有 VisionOS 资源警告和 "Closing analysis cache, 0.00 MiB read, 0.00 MiB written"，说明它根本没读 trace。
- **UE 5.7**: CLI auto-export feature appears to be broken or removed
- **结论**: UnrealInsights CLI 的自动导出功能在 UE 5.4 和 5.7 上都不工作，文档描述的功能可能是早期版本的，或者需要特殊触发条件（未知）

## Solution
**Use the UnrealInsights GUI on Windows directly** — the GUI is the only reliable way to export timing data from trace files.

### Manual Export Steps (UE 5.7)
1. Open `UnrealInsights.exe` from `D:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\`
2. Drag .utrace file into the window (or File → Open)
3. Click **Timing** tab
4. **File → Export Timer Statistics**
5. Save as CSV

### Alternative: Install UE 5.4
~~If CLI auto-export is needed, install UE 5.4 (confirmed to support `-ExecOnCompleteCmd`).~~ **UE 5.4 也不行（2026-04-27 实测）**。所有参数组合均失败，和 UE 5.7 一样不分析 trace 文件。

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
- UE 5.4: `D:\Program Files\Epic Games\UE_5.4\Engine\Binaries\Win64\UnrealInsights.exe`（已安装，2026-04-27）
- UE 5.7: `D:\Program Files\Epic Games\UE_5.7\Engine\Binaries\Win64\UnrealInsights.exe`
- WSL UE 5.4: `/mnt/d/Program Files/Epic Games/UE_5.4/Engine/Binaries/Win64/UnrealInsights.exe`
- Trace 文件目录: `D:\ZY_Files\trace文件\`（含中文，注意路径处理）

## Windows 拖拽导出脚本（bat）— ⚠️ UE 5.7 不支持

在用户桌面放置 `utrace_to_csv.bat`，可直接拖拽 .utrace 文件导出 CSV。

### ⚠️ 已知坑点

1. **UE 5.7 CLI 不支持自动导出**: 所有 `-ExecOnCompleteCmd` 参数组合均失败，脚本无法工作
2. **中文路径问题**: trace 文件路径含中文字符时，CMD 的 `copy`/`dir` 等命令可能报"系统找不到指定的路径"
   - 解决方案：用 `pushd` 先进入中文目录，再用纯英文文件名操作
   - 或用 `chcp 65001` 设置 UTF-8 代码页
3. **bat 文件编码（WSL 写入）**: WSL 的 `write_file` 或 `cat > file.bat` 默认生成 UTF-8 + LF 换行
   - **必须转换为 CRLF**：`sed -i 's/$/\r/' file.bat` 或 `unix2dos file.bat`
   - 否则 CMD 会把每行拆成多个命令执行，报 `'local' 不是内部或外部命令` 等奇怪错误
   - 脚本中含中文时还要加 `chcp 65001 >nul`
4. **WSL 写 bat 文件的标准流程**:
   ```bash
   cat > /mnt/c/Users/xxx/Desktop/script.bat << 'EOF'
   @echo off
   chcp 65001 >nul
   ... content ...
   EOF
   sed -i 's/$/\r/' /mnt/c/Users/xxx/Desktop/script.bat
   ```
5. **用户要求**: 不要修改任何 Windows 软件设置，脚本只做只读+导出

### 推荐脚本（UE 5.4，支持中文路径）

```bat
@echo off
chcp 65001 >nul
echo ============================================
echo  UE 5.4 CLI CSV Export
echo ============================================

if not exist "C:\ue_export_temp" mkdir "C:\ue_export_temp"

echo [Step 1] Copy trace file to temp dir...
pushd "D:\ZY_Files\trace文件\2026.01\01.21"
copy "20260120_182108.utrace" "C:\ue_export_temp\trace.utrace"
popd
if errorlevel 1 (
    echo ERROR: Failed to copy trace file!
    pause
    exit /b 1
)
echo Copy OK.

echo [Step 2] Write RSP file...
echo TimingInsights.ExportTimerStatistics C:\ue_export_temp\TimerStats.csv > "C:\ue_export_temp\export_commands.rsp"

echo [Step 3] Run UnrealInsights...
"D:\Program Files\Epic Games\UE_5.4\Engine\Binaries\Win64\UnrealInsights.exe" -OpenTraceFile="C:\ue_export_temp\trace.utrace" -NoUI -AutoQuit -ExecOnCompleteCmd="@C:\ue_export_temp\export_commands.rsp"

echo.
echo [Step 4] Check results:
if exist "C:\ue_export_temp\TimerStats.csv" (
    echo SUCCESS! CSV exported.
) else (
    echo FAILED. No CSV found.
)
pause
```

**关键点**：
- UE 路径用 UE_5.4（不是 5.7）
- `chcp 65001` 处理中文
- `pushd` 进入中文目录再 copy，避免中文路径问题
- WSL 写完后必须 `sed -i 's/$/\r/' file.bat` 转 CRLF

## Known Compatible Traces
- `20260423_163023.utrace` (1.2GB, Windows UE 5.4, Infinity Nikki/X6Game) — opens fine
- Mac-recorded traces may have protocol version mismatches on Windows
