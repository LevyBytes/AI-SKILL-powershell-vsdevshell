# Windows PowerShell 5.1 Compatibility

Windows PowerShell 5.1 (Desktop edition, `powershell.exe`) and PowerShell 7+ (`pwsh.exe`) are different products that install side by side. A repository that targets Windows PowerShell 5.1 needs syntax that parses and runs under `powershell.exe`. Validate against that engine, not against `pwsh`.

Declare the requirement at the top of scripts when it matters:

```powershell
#Requires -Version 5.1
#Requires -PSEdition Desktop
```

`#Requires -Version 5.1` declares a minimum version, not an exact engine; PowerShell 7 also satisfies the version requirement. Add `-PSEdition Desktop` when Windows PowerShell is required. Requirements are checked globally before execution even when placed later in the file; an exact 5.1 runtime requirement needs an explicit version check as well.

## Syntax unavailable in Windows PowerShell 5.1 and the 5.1-safe alternative

These constructs are parse errors or missing features in Windows PowerShell 5.1. Each row gives the safe replacement.

| Newer PowerShell construct | 5.1-safe alternative |
| --- | --- |
| Ternary `$x = $a ? $b : $c` | `if ($a) { $x = $b } else { $x = $c }` |
| Null-coalescing `$x = $a ?? $b` | `$x = if ($null -ne $a) { $a } else { $b }` |
| Null-coalescing assignment `$a ??= $b` | `if ($null -eq $a) { $a = $b }` |
| Null-conditional access `${a}?.Prop` / `?[0]` | explicit `if ($null -ne $a) { $a.Prop }` |
| Pipeline chains `cmd1 && cmd2` / `cmd1 \|\| cmd2` | After a native call, capture `$code = $LASTEXITCODE`; run the success branch for `$code -eq 0`, the failure branch for `$code -ne 0`. For cmdlets, capture `$?` immediately instead. |
| `ForEach-Object -Parallel` | sequential `foreach` loops; background jobs only when genuinely required |
| `Join-String` cmdlet | the `-join` operator (present in 5.1): `$items -join ', '` |
| `` `e `` and `` `u{...} `` escape sequences | not available in 5.1 (about_Special_Characters 5.1 documents `` `0 `a `b `f `n `r `t `v `` only) |
| Assuming `pwsh` exists | target `powershell.exe`; check `$PSVersionTable.PSVersion` and `PSEdition` |

The 5.1 language does not include ternary, `??`, `??=`, `&&`/`||` chains, `` `e ``, or `` `u{} ``; those arrived in PowerShell 6/7.

## Behavior differences worth knowing in 5.1

- Default file encodings for `>`/`Out-File`/`Set-Content` differ between Windows PowerShell 5.1 and PowerShell 7. Never rely on the default; always pass `-Encoding` explicitly (see [files paths and encoding](files-paths-and-encoding.md)).
- Until PowerShell 7, wrapping a command in parentheses `(...)`, `$(...)`, or `@(...)` resets `$?` to True even when the inner command failed. Capture `$?` or `$LASTEXITCODE` before wrapping anything.
- `$PSScriptRoot` is valid in all scripts since PowerShell 3.0; safe in 5.1.
- Real-to-integer casts round to nearest even ("banker's rounding"), and string conversions generally use the invariant culture.

## Parser validation pattern

Parse 5.1-target scripts with the Windows PowerShell engine before running them:

```powershell
$tokens = $null
$errors = $null
[System.Management.Automation.Language.Parser]::ParseFile('<WORK_ROOT>\script.ps1', [ref]$tokens, [ref]$errors) | Out-Null
if ($errors.Count -ne 0) {
    $errors | ForEach-Object { [Console]::Error.WriteLine(('{0} at line {1}' -f $_.Message, $_.Extent.StartLineNumber)) }
    exit 1
}
```

Run the validator inside `powershell.exe` (for example `& "$Env:windir\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -File <validator>.ps1`) so the 5.1 parser does the parsing. A PS7-only construct can parse cleanly under `pwsh` and still fail under 5.1.

## Community script style (practical tip)

- Explicit `param(...)` blocks; `[CmdletBinding()]` for advanced functions.
- `$ErrorActionPreference = 'Stop'` near the top when failures must stop the script.
- Small helper functions, phase-marker logging, deterministic final status line.
- Placeholders such as `<REPO_ROOT>`, `<WORK_ROOT>`, `<REPORTS_ROOT>` in reusable documentation.
- ASCII-safe punctuation in generated strings: PowerShell treats smart quotes as quote characters, so copied typographic punctuation can change parsing.

## Primary sources

[Version requirements](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_requires?view=powershell-5.1) | [Changes from Windows PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/whats-new/differences-from-windows-powershell?view=powershell-7.6)
