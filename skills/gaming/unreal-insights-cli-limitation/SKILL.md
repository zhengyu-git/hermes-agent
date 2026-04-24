---
name: unreal-insights-cli-limitation
description: UnrealInsights CLI cannot export CSV in WSL — must use Windows GUI
tags: [unreal-engine, unreal-insights, wsl, performance-analysis, trace-analysis]
---

# Unreal Insights CLI Limitation in WSL

## Problem
UnrealInsights CLI via WSL cannot export CSV/Timing data — the `-ExecOnAnalysisCompleteCmd` command never executes, even with `-NoUI -AutoQuit -WaitForAnalysis`.

## Symptom
- UnrealInsights opens the trace file successfully
- Log shows analysis completing ("Closing analysis cache")
- But no CSV files are produced
- Exit code is 0

## Verified Parameters That Do NOT Work
- `-ExecOnAnalysisCompleteCmd="TimingInsights.ExportTimerStatistics ..."`
- Response file with multiple commands (`@/path/to/export.rsp`)
- `-WaitForAnalysis` flag
- Combinations of all the above

## Root Cause
WSL environment causes path resolution and working directory issues. UnrealInsights runs but cannot write output to Windows paths from WSL context.

## Solution
**Use the UnrealInsights GUI on Windows directly** — double-click the .utrace file or open it manually in the Windows GUI application. The GUI is the only reliable way to export timing data from trace files when running via WSL.

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
- Windows: `D:\Program Files\Epic Games\UE_5.4\Engine\Binaries\Win64\UnrealInsights.exe`
- WSL: `/mnt/d/Program Files/Epic Games/UE_5.4/Engine/Binaries/Win64/UnrealInsights.exe`

## Known Compatible Traces
- `20260423_163023.utrace` (1.2GB, Windows UE 5.4, Infinity Nikki/X6Game) — opens fine
- Mac-recorded traces may have protocol version mismatches on Windows
