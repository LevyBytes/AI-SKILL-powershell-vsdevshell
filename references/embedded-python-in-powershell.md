# Embedded Python in PowerShell

Generating Python from PowerShell breaks when quoting and interpolation are casual. Use single-quoted line arrays, inject values through argv/env/JSON files instead of string interpolation, and validate with `py_compile` before running.

## Safe line-array pattern

Single-quoted PowerShell strings are verbatim - PowerShell will not expand `$name`, backticks, or subexpressions inside them, so Python code survives intact:

```powershell
$py = @(
    'import json',
    'import sys',
    'from pathlib import Path',
    'print("START generated job")',
    'config = json.loads(Path(sys.argv[1]).read_text(encoding="utf-8-sig"))',
    'print("STATUS OK")'
)
Set-Content -LiteralPath '<WORK_ROOT>\generated.py' -Value $py -Encoding UTF8
```

Double a single quote for the PowerShell string layer; it then appears once in the generated text. The resulting Python still needs valid Python quoting. For example, the PowerShell literal `'print("it''s ready")'` produces valid Python `print("it's ready")`. Preserve indentation and line boundaries for compound statements; do not interpolate data into source.

## Injecting values safely

Never interpolate values into Python source. In order of preference:

1. Command-line arguments: `& $python '<WORK_ROOT>\generated.py' $inputPath $mode` and read `sys.argv` in Python.
2. A JSON input file written by PowerShell and read by Python (`encoding='utf-8-sig'` tolerates the 5.1 BOM - practical tip).
3. Environment variables: `$Env:JOB_INPUT = $inputPath` then `os.environ['JOB_INPUT']`.

`-f` formatting is still source interpolation; it does not escape arbitrary values for Python. Keep source fixed and pass data through the documented input channel. PowerShell 5.1 native argument serialization can alter quotes or empty values, so JSON input is preferable for arbitrary text.

## Validation and invocation

```powershell
& '<TOOLS_ROOT>\python.exe' -m py_compile '<WORK_ROOT>\generated.py'
if ($LASTEXITCODE -ne 0) { throw ('py_compile failed: {0}' -f $LASTEXITCODE) }

& '<TOOLS_ROOT>\python.exe' '<WORK_ROOT>\generated.py' '<WORK_ROOT>\job-config.json'
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) { throw ('Generated job failed: {0}' -f $exitCode) }
```

- `python -m py_compile <file>` exits nonzero when any file fails to compile - a cheap syntax gate with no side effects beyond `__pycache__`.
- Python commonly exits 1 for uncaught exceptions and 2 for command-line usage errors; a script can define other codes. Check the job contract and process-start failures too.
- For redirected Python standard streams, set `PYTHONIOENCODING=utf-8` and configure the reader for UTF-8 too. Modern Windows console buffers have separate handling; do not treat that environment variable as a universal console switch.

## Encoding both directions

- PowerShell 5.1 `Set-Content -Encoding UTF8` writes a BOM. Python source files tolerate a UTF-8 BOM, and readers should use `utf-8-sig` for data files PowerShell wrote.
- Files Python writes as `utf-8` have no BOM. In Windows PowerShell 5.1 use `Get-Content -Encoding UTF8`; the default ANSI decoding can corrupt non-ASCII text.
- Use the BOM-less .NET writer (see [files paths and encoding](files-paths-and-encoding.md)) when a consumer rejects BOM.

For stdout/stderr capture under a Stop error preference, use [direct process capture](native-process-output-capture.md).

Keep generated scripts and bytecode caches in the caller-designated output location.

## Primary sources

[PowerShell quoting](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_quoting_rules?view=powershell-5.1) | [Python source compilation](https://docs.python.org/3/library/py_compile.html) | [Python standard-stream encoding](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONIOENCODING)
