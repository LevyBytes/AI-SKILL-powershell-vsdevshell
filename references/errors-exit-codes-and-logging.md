# Errors, Exit Codes, and Logging

Open when handling PowerShell cmdlet errors, native process status, cleanup, or execution logs in Windows PowerShell 5.1.

## Terminating and non-terminating errors

`try`/`catch` catches terminating errors. Use a cmdlet's `-ErrorAction Stop` when a non-terminating error must enter `catch`, or set `$ErrorActionPreference = 'Stop'` for the intended scope. A command's common parameter overrides that preference for the command.

Inside `catch`, `$_` is an ErrorRecord with `Exception.Message` and `ScriptStackTrace`. Put specific exception catches before a general catch. `throw` terminates; `Write-Error` is non-terminating by default but respects error preferences.

This complete input-check template prints a failure and exits nonzero when the required file is absent:

```powershell
try {
    Get-Item -LiteralPath '<REPO_ROOT>\required.file' -ErrorAction Stop | Out-Null
} catch {
    [Console]::Error.WriteLine(('Missing required file: {0}' -f $_.Exception.Message))
    exit 1
} finally {
    Write-Host 'Input-verification phase finished.'
}
```

`finally` runs during normal unwinding, including an `exit` from the try/catch. It is not a guarantee after forced process termination. A cleanup message in `finally` must not claim that the operation succeeded.

## Native failures use a separate contract

For a successfully launched native executable:

```powershell
& $exe @argv
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) {
    throw ('Command failed with exit code {0}: {1}' -f $exitCode, $exe)
}
```

Bind `$exe` to the intended native executable and `$argv` to its arguments before this fragment. Handle command-resolution and process-start failures separately; they cannot be diagnosed by a stale exit code. Cmdlet error preferences do not generally turn native nonzero exit codes into exceptions in 5.1.

Redirected native stderr can become `NativeCommandError` under `$ErrorActionPreference = 'Stop'` even when the child exits zero. Both `2>&1` and `2> file` can trigger it. Use the [complete direct-process capture example](native-process-output-capture.md) to keep stdout, stderr and exit status separate.

## Tool exit conventions

- Python commonly exits 1 for an uncaught exception and 2 for command-line usage errors. A script may choose other codes with `sys.exit`; check its own contract.
- `python -m py_compile` returns nonzero for compilation errors and writes bytecode cache files; put those products in the assigned work area.
- Cargo build documents 0 for success and 101 for failure. npm scripts default to `cmd.exe` on Windows, unless `script-shell` is configured, and failing scripts produce nonzero status.
- An output left by a previous run cannot establish success for a failed current run.

## Phase logging

This helper requires an existing writable work directory. Its invocation lines illustrate log format; emit final success only after the actual task's checks pass.

```powershell
$script:TaskLogPath = Join-Path '<WORK_ROOT>' 'task.log'
function Write-Phase {
    param([Parameter(Mandatory=$true)][string]$Message)
    $line = ('[{0}] {1}' -f (Get-Date -Format 'yyyy-MM-dd HH:mm:ss'), $Message)
    Write-Host $line
    Add-Content -LiteralPath $script:TaskLogPath -Value $line -Encoding UTF8
}

Write-Phase 'START task-name'
Write-Phase 'PHASE validate-inputs'
```

Record inputs, output locations, the selected executable, argument values and captured exit codes. Follow the caller's status schema; labels such as `VALIDATED`, `FAILED_WITH_EVIDENCE`, `SKIPPED_BY_SCOPE`, `BLOCKED_WITH_EVIDENCE` and `NOT_ATTEMPTED` are optional examples, not built-in PowerShell states. Do not label a skipped check as a user preference unless the user actually requested the skip.

## Related

[Native quoting](quoting-and-native-commands.md) · [Output capture](native-process-output-capture.md) · [Validation](validation-and-ast-checks.md) · [Encoding](files-paths-and-encoding.md)
