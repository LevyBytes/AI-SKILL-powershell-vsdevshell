# Validation and AST Checks

Validate the thing the task changed before claiming success, and validate it with the engine that will run it.

## PowerShell AST parse

Use the Windows PowerShell parser for Windows PowerShell 5.1 scripts, executed under `powershell.exe`:

```powershell
$tokens = $null
$errors = $null
[System.Management.Automation.Language.Parser]::ParseFile('<WORK_ROOT>\script.ps1', [ref]$tokens, [ref]$errors) | Out-Null
if ($errors.Count -ne 0) {
    $errors | ForEach-Object { [Console]::Error.WriteLine(('{0} at line {1}' -f $_.Message, $_.Extent.StartLineNumber)) }
    exit 1
}
```

List every parse error with its line number; writing diagnostics to Console.Error avoids a Stop preference aborting the diagnostic loop at its first Write-Error. A clean parse under `pwsh` proves nothing about 5.1 - run the validator with the target engine.

## Smoke checks

After a clean parse, run the cheapest real execution the script supports:

- `-Help` or usage mode for wrapper scripts (assert exit code 0 and expected usage text).
- `-WhatIf`, dry-run, or plan modes where implemented.
- A minimal no-op invocation otherwise.

Parsing and bounded smoke checks catch syntax and exercised behavior; they do not establish every runtime branch, native argument boundary or output encoding. Use focused fixtures for those contracts. Confirm a no-op/help mode exists before invoking it.

## JSON and JSONL deserialization

These Windows PowerShell 5.1 examples check required input and deserialization. `ConvertFrom-Json` is permissive: success does not establish strict JSON grammar, unique keys, a schema or required application fields. Use a strict parser and application checks when those contracts matter. See [ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json?view=powershell-5.1).

```powershell
$json = Get-Content -LiteralPath '<REPORTS_ROOT>\out.json' -Raw -Encoding UTF8 -ErrorAction Stop
if ([string]::IsNullOrWhiteSpace($json)) { throw 'Required JSON input is empty' }
try { $null = $json | ConvertFrom-Json -ErrorAction Stop }
catch { throw ('Cannot deserialize JSON: {0}' -f $_.Exception.Message) }
```

```powershell
$lines = @(Get-Content -LiteralPath '<REPORTS_ROOT>\records.jsonl' -Encoding UTF8 -ErrorAction Stop)
if ($lines.Count -eq 0) { throw 'Required JSONL input is empty' }
for ($index = 0; $index -lt $lines.Count; $index++) {
    $lineNo = $index + 1
    if ([string]::IsNullOrWhiteSpace($lines[$index])) {
        throw ('Empty JSONL record at line {0}' -f $lineNo)
    }
    try { $null = $lines[$index] | ConvertFrom-Json -ErrorAction Stop }
    catch { throw ('Cannot deserialize JSONL at line {0}: {1}' -f $lineNo, $_.Exception.Message) }
}
```

Read failures stop before parsing. Exceptions must fail the calling validation step; do not catch them and then report success. These examples reject blank JSONL records, as required by [JSON Lines](https://jsonlines.org/), but retain the parser limits described above.

Validate cross-references and required fields separately. Windows PowerShell 5.1 writes a BOM with `-Encoding UTF8`; use the BOM-less writer in [file encoding](files-paths-and-encoding.md) for strict consumers.

## Native tools and outputs

- Check `$LASTEXITCODE` for every native command; record command, arguments, and exit code.
- `python -m py_compile` for generated Python; build/test exit codes for Rust (`cargo` fails with 101) or C++ (MSBuild/`cl.exe`/`link.exe`) when in scope.
- Confirm expected output paths exist, are non-empty where appropriate, and landed in the repo-designated artifact roots.
- Use noninteractive stdin redirection only for known prompt blockers, documented.

## What not to validate

Keep validation within the task and repository instructions. These examples do not require Git checks or establish Git cleanliness. If a check was excluded by the user, say so; if it was simply unnecessary or not run, report that actual reason instead of inventing a user preference.
