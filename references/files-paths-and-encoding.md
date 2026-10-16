# Files, Paths, and Encoding

Windows scripts should treat paths and encodings as explicit design choices, never defaults.

## Path handling

```powershell
$outDir = Join-Path '<WORK_ROOT>' 'validation'
if (-not (Test-Path -LiteralPath $outDir)) {
    New-Item -ItemType Directory -Path $outDir -Force | Out-Null
}
```

- `Join-Path` for composition; `Test-Path -LiteralPath` for existence; `Get-Item` for existing items.
- `Resolve-Path` only for paths that already exist - never for output files not yet created.
- `-LiteralPath` whenever a path may contain wildcard characters (`[`, `]`, `*`, `?`).
- Prefer absolute paths in agent scripts. Quote any path containing spaces.
- Setting env vars for child tools: `$Env:NAME = 'value'` affects the current process and children only; persistence needs `[Environment]::SetEnvironmentVariable(...,'User'|'Machine')`. Route tool TEMP/TMP to the approved work root when the repo requires it.

## Long paths

Windows path APIs historically limit fully qualified paths to roughly 260 characters; deep tool output trees can exceed it and fail with misleading errors. Practical tip:

- Keep artifact roots short (for example `X:\work` rather than deeply nested folders).
- Watch for failures that only reproduce on long paths; shorten the root rather than reaching for `\\?\` prefixes, which many tools and cmdlets do not handle uniformly.
- When a long path is unavoidable, verify the consuming tool supports it before relying on it.

## Encoding in Windows PowerShell 5.1

Defaults differ by cmdlet and differ from PowerShell 7. Always pass `-Encoding` explicitly:

```powershell
Set-Content -LiteralPath '<WORK_ROOT>\report.md' -Value $lines -Encoding UTF8
Add-Content -LiteralPath '<WORK_ROOT>\task.log' -Value $message -Encoding UTF8
```

- In 5.1, `-Encoding UTF8` writes UTF-8 WITH a byte order mark (BOM). For BOM-less UTF-8 input, also specify `Get-Content -Encoding UTF8`; default reads use the active ANSI code page when no BOM is present.
- PowerShell 7 defaults to BOM-less UTF-8; the same script can produce different bytes on different engines - one more reason to be explicit.
- Redirection operators (`>`) behave like `Out-File` with no parameters; use `Out-File -Encoding ...` when the byte format matters.

BOM-less UTF-8 when a consumer rejects BOM (.NET, clearly marked as the explicit escape hatch):

```powershell
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[System.IO.File]::WriteAllText('<WORK_ROOT>\out.json', $text, $utf8NoBom)
```

Encoding does not select line endings. If an output requires LF, join lines explicitly with ``"`n"`` and write that string through the .NET writer; `Set-Content` line-array output normally uses Windows line endings. Non-ASCII PowerShell 5.1 script source needs a detectable encoding (commonly a UTF-8 BOM); ASCII-only source avoids that ambiguity.

Practical tip: JSON written by 5.1 (`ConvertTo-Json | Set-Content -Encoding UTF8`) carries a BOM that python `json.loads` rejects when read as plain `utf-8`. Either write BOM-less via .NET, or have Python readers use `encoding='utf-8-sig'` (tolerates BOM presence and absence).

Use the output locations and cleanup rules specified by the caller or repository.

## Primary sources

[Character encoding](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_character_encoding?view=powershell-5.1) | [Windows path limits](https://learn.microsoft.com/en-us/windows/win32/fileio/maximum-file-path-limitation)
