---
name: writing-windows-scripts
description: Use when writing or editing Windows .bat/.cmd/.ps1 scripts — especially with Chinese or other non-ASCII text — before delivering or committing. Prevents cmd parser explosions ("xxx is not recognized as an internal or external command" mid-line fragments), mojibake 乱码, and scripts that break only on the user's machine.
---

# Writing Windows Scripts (.bat / .ps1)

## Overview

cmd.exe parses batch files byte-by-byte and is encoding/line-ending sensitive: a .bat that looks correct in the editor can explode at runtime. PowerShell 5.1 and pwsh 7 read script files with different encoding defaults. **Static review proves nothing — only an actual run on the target shell proves a script works.**

## The One Rule

Never deliver a script you haven't run. If the full script is destructive (kills processes, uninstalls), extract harmless fragments (the encoding-sensitive lines, the registry queries, the echo/choice lines) and run those. An agent that "statically verified" a batch file has verified nothing — baseline failure observed: confident report, file exploded on first real run.

## Encoding & Line Endings (hard requirements)

| File | Line endings | Encoding | Notes |
|------|-------------|----------|-------|
| `.bat` / `.cmd` | **CRLF mandatory** | ASCII-only preferred | LF-only → cmd misaligns lines → `'127.0.0.1' is not recognized` style fragment errors. Symptoms look like an encoding bug; root cause is LF. |
| `.bat` with Chinese (if unavoidable) | CRLF | UTF-8 **no BOM** + line 1 `@echo off & chcp 65001 >nul` | BOM at byte 0 can break line 1 on older cmd. Verified working on Win11 with CRLF. |
| `.bat` GBK-encoded | — | **never** | GBK trail bytes can be `(` `)` (0x28/0x29) → breaks parenthesized blocks. Displays as mojibake under a 65001 console. |
| `.ps1` with non-ASCII | CRLF | UTF-8 **with BOM** | PS 5.1 reads BOM-less UTF-8 as ANSI → mojibake. pwsh 7 tolerates no-BOM, so "works in pwsh" proves nothing for PS 5.1. |

Editors and file-writing tools often emit LF + no BOM. Convert after writing (PowerShell):

```powershell
$raw = [IO.File]::ReadAllText($f, [Text.Encoding]::UTF8)
[IO.File]::WriteAllText($f, ($raw -replace "(?<!`r)`n","`r`n"), [Text.UTF8Encoding]::new($isPs1))  # $true = BOM
```

Repo defense: add `*.bat text eol=crlf` and `*.ps1 text eol=crlf` to `.gitattributes`.

## Preferred Architecture

Nontrivial logic + Chinese UI → split:

- `.bat` = **ASCII-only** thin launcher: `powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0foo.ps1"` — zero parser risk, double-click friendly.
- `.ps1` = all logic and all non-ASCII text, UTF-8 with BOM.

## Never Hardcode System Facts

Don't write install paths / registry key names from docs or assumptions — installers change them per version. Observed real case: electron-builder v25 writes the uninstall key as a **GUID** (not the appId), leaves `InstallLocation` **empty**, and installs to a **lowercase** dir (`Programs\lingdoc-desktop`, not `Programs\LingDoc`). Inspect the target machine first:

```powershell
Get-ChildItem 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall',
              'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall' |
  ForEach-Object { Get-ItemProperty $_.PSPath } |
  Where-Object DisplayName -like '*AppName*' |
  Select-Object DisplayName, UninstallString, QuietUninstallString  # parse exe path from these
```

Same rule for data dirs: don't assume one canonical location. App data scatters across `%LOCALAPPDATA%`, `%APPDATA%`, `%USERPROFILE%` under multiple name variants (`LingDoc`, `lingdoc-desktop`, `.lingdoc`, `lingdoc-desktop-updater`) — glob all roots for name variants and show per-dir sizes so the user sees what deletion actually removes. Observed on one machine: 6 distinct data dirs, including a 221MB updater cache invisible to anyone who checked only `%APPDATA%`.

Destructive scripts benefit from a built-in dry-run flag (`-WhatIf` param that prints instead of executes) — it lets you run the full script end-to-end on the real machine as your verification, without the destructive effect. Agents that did this caught runtime bugs static review missed (StrictMode property access, $null member access).

## Verification Checklist (before delivering)

- [ ] Byte check: CRLF everywhere (`(?<!\r)\n` count = 0); .ps1 starts `EF BB BF`; .bat has no BOM
- [ ] Syntax check .ps1 with **powershell.exe 5.1**, not only pwsh: `PSParser::Tokenize(...)` error count = 0
- [ ] Ran the script or its harmless fragments on the real target shell (cmd.exe for .bat)
- [ ] Destructive steps (delete data, kill processes) are behind explicit user confirmation

## Common Mistakes

| Mistake | Reality |
|---------|---------|
| "我静态检查过了，括号配平、无乱码" | cmd 的 LF/编码 bug 编辑器里不可见，只有运行能暴露 |
| UTF-8 BOM 写进 .bat | 部分 cmd 版本首行直接报错；.bat 永远无 BOM |
| .ps1 无 BOM，pwsh 里测试通过 | PS 5.1（双击默认引擎）按 ANSI 读 → 中文全乱 |
| 按文档写死注册表键名/安装路径 | 实测机器才是真相：GUID 键、空 InstallLocation、小写目录都出现过 |
| 依赖用户机器 console 代码页（GBK  bat） | 65001 控制台下 GBK 立即乱码；chcp 行为随机器变 |
