---
name: ue5-log-analysis
description: Analyze Unreal Engine 5 game logs for crashes, errors, and performance issues
---
# UE5 Game Log Analysis

## When to Use
Analyzing Unreal Engine 5 game logs (.txt) for crashes, performance issues, or errors.

## File Access
Game logs on Windows are typically at:
```
C:\Users\<username>\Desktop\LOG\
```
WSL path: `/mnt/c/Users/<username>/Desktop/LOG/`

## Key Search Patterns

### Crash/Fatal Errors
```bash
grep -i "Crash\|FATAL\|Assertion\|Exit" logfile.txt
grep -i "stack trace\|dump" logfile.txt
grep -i "ensure\|verify\|fatal" logfile.txt
```

### PSO (Pipeline State Object) Failures
```bash
grep -i "PSO.*compile failure" logfile.txt | wc -l  # Count failures
grep -i "LogRHI: Error" logfile.txt                 # RHI errors
grep -i "LogMetal: Error" logfile.txt                # Metal errors
```

### Metal/GPU Errors (Mac)
```bash
grep -i "LogMetal: Error" logfile.txt               # Metal API errors
grep -i "metalmap" logfile.txt                      # Missing metalmap files
grep -i "MTLPixelFormatInvalid" logfile.txt          # Pixel format issues
```

### Performance Metrics
```bash
grep -i "FPS\|frame\|Stat\|stat unit" logfile.txt
grep -i "LogStats" logfile.txt
```

### System Info (device profile, GPU, OS)
```bash
grep -i "LogInit:.*DeviceId\|GPU\|MacBook\|chip" logfile.txt
grep -i "Selected Device Profile" logfile.txt
```

### Memory Issues
```bash
grep -i "OutOfMemory\|OOM\|VRAM\|memory" logfile.txt
```

### CrashSight Plugin (UE crash reporting)
```bash
grep -i "CrashSight\|Session CrashGUID" logfile.txt
```

## UE5 Log Structure
- Line 1: `Log file open, MM/DD/YY HH:MM:SS`
- Contains: `[YYYY.MM.DD-HH.MM.SS:mmm][thread]` prefix
- `LogXXX: Error/Warning/Display` format

## Common Error Types
| Error | Meaning |
|-------|---------|
| `PSO compile failure` | Shader/render pipeline compilation failed |
| `MTLPixelFormatInvalid` | Metal pixel format issue (Mac) |
| `No .metalmap file found` | Missing Metal shader mapping |
| `Failed to create graphics pipeline` | RHI can't create GPU pipeline |
| `MTLPixelFormatInvalid` | Metal pixel format mismatch (Mac GPU) |
| `Shaders reads from a color attachment` | PSO shader color attachment format error |
| `ensure() failed` | Assertion failure in code |
| `FATAL` | Unrecoverable error |
| `Graphics PSO (XXX) compile failure` | Count with `| wc -l` for total PSO failures |

## Device Profile (Mac)
```
LogInit: OS: macOS XX.X.X, CPU: Apple M1/M2/M3, GPU: Apple M1/M2/M3
LogRHI: RHI Name: Metal
LogRHI: RHI ShaderFormat: SF_METAL_SM5
LogMetal: VRAM (MB): XXXXX
```

## Crash Detection Tips
- No `FATAL` or crash dump in log + sudden end = process was likely **force-killed** (OOM/系统终止)
- Log ends with errors but no crash dump = crash happened but graceful exit didn't occur
- Check `tail -50 logfile.txt` for end-of-log patterns

## Real Case Study: Infinity Nikki (Mac M2)
Log file: `X6Game(2).txt` (~8.6MB, 128k lines)
- Device: MacBook M2, macOS 15.7.2, VRAM 12GB
- Game: X6Game (Infinity Nikki), UE5 Rel_2.5
- **283 PSO compilation failures** - count with `grep -i "PSO.*compile failure" logfile.txt | wc -l`
- Fatal error: `MTLPixelFormatInvalid` - Metal pixel format mismatch
- Log ended suddenly at 14:43:51 with NO crash dump → **process was force-killed**
- Key insight: Metal errors + no crash dump = OS terminated the process, not UE5 crash

## Unreal Insights .utrace Files
Trace files on Windows: `C:\Users\<username>\Desktop\Trace\`
Launch: `"<UnrealInsights.exe path>" "<trace file path>"`
Example:
```bash
"/mnt/d/Program Files/Epic Games/UE_5.4/Engine/Binaries/Win64/UnrealInsights.exe" "/mnt/c/Users/xxx/Desktop/Trace/20260413_170922.utrace"
```
- Use `background=true` to launch
- Verify with `ps aux | grep -i UnrealInsights`

## Log Size Note
Large logs (~100k+ lines) may need offset pagination:
```python
read_file(path, offset=128200, limit=100)
```
Or use `tail`:
```bash
tail -100 "/mnt/c/Users/xxx/Desktop/LOG/X6Game.txt"
```
