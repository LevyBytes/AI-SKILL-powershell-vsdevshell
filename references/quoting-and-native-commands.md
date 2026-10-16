# Quoting and Native Commands

Open when invoking a Windows native executable from PowerShell 5.1 with paths, argument arrays, literal quotes, or explicit exit-code handling.

## Separate the executable from its arguments

This is an intentionally failing example: the call operator treats the whole string as a command name, not a command line to split.

```powershell
$commandText = '"<TOOLS_ROOT>\tool.exe" --input "file.txt"'
& $commandText
```

Use the executable path plus separate argument values for ordinary paths and switches:

```powershell
$exe = '<TOOLS_ROOT>\tool with spaces.exe'
$argv = @('--input', '<WORK_ROOT>\input file.txt', '--verbose')
& $exe @argv
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) { throw ('tool failed: {0}' -f $exitCode) }
```

The example assumes the native executable starts successfully. Resolve the intended executable first; a PowerShell command-resolution or launch error is not a new native exit code. Do not use a stale `$LASTEXITCODE` as evidence that a failed launch succeeded.

## PowerShell 5.1 argument limits

Argument arrays keep PowerShell expressions separate, but Windows PowerShell 5.1 then serializes them into a native command line. Empty strings can disappear, embedded quotes can be removed, and quotes/backslashes can change boundaries seen by the child. Spaces in ordinary file arguments generally work; this is not an arbitrary-data transport guarantee.

When values include quotes, empty strings or trailing backslashes, test the actual received arguments with that executable. Prefer a JSON/config file, response file or documented stdin protocol for complex data. Response-file support and syntax belong to the target tool. PowerShell 7.3's newer argument-passing behavior is not a 5.1 feature.

## Build-tool examples

These are independent command templates; resolve the executable and paths for the intended environment. Record the exit code immediately after each call when adapting a template into a script.

```powershell
& '<TOOLS_ROOT>\python.exe' '<WORK_ROOT>\script.py' '--mode' 'validate'
$pythonExit = $LASTEXITCODE

& '<TOOLS_ROOT>\gui-tool.exe' '--background' '--python' '<WORK_ROOT>\job.py'
$guiExit = $LASTEXITCODE

& cargo.exe build --release --manifest-path '<REPO_ROOT>\Cargo.toml'
$cargoExit = $LASTEXITCODE

& npm.cmd run build
$npmExit = $LASTEXITCODE

$msbuildArgs = @('<REPO_ROOT>\Project.sln', '-m', '-t:Build', '-p:Configuration=Release', '-p:Platform=x64')
& msbuild.exe @msbuildArgs
$msbuildExit = $LASTEXITCODE
```

`npm.cmd` selects the Windows launcher explicitly; package scripts use `cmd.exe` by default, configurable through npm's `script-shell`. MSBuild list switches containing semicolons or commas must remain one quoted PowerShell argument. See [build tools](msbuild-msvc-build-tools.md).

## Stop parsing with `--%`

For Windows native commands, `--%` stops PowerShell interpretation until the next newline or pipe. `%NAME%` environment references still expand; `$variables` do not. A semicolon or continuation backtick cannot end or extend its effect. Redirection tokens following it become literal arguments.

The following is an ACL-changing command template, not a harmless quoting test; use it only when that permission change is the task:

```powershell
icacls X:\VMS --% /grant Dom\HVAdmin:(CI)(OI)F
```

The receiving executable still parses its command line. Stop-parsing does not supply a universal literal-quote escape rule.

## Exit status and stderr

Capture `$LASTEXITCODE` immediately after a successfully launched native program. `$?` is PowerShell's last-operation status; parentheses, subexpressions and array expressions reset it in 5.1. Scripts do not all update `$LASTEXITCODE` identically; use the invoked script's exit contract.

Stderr may contain progress on a successful run. In 5.1, redirected stderr can become `NativeCommandError` under a Stop preference even with separate output files. Use [direct process output capture](native-process-output-capture.md) when this distinction matters. Do not add stdin piping unless the tool's documented protocol requires it.

## Primary sources

[Quoting rules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_quoting_rules?view=powershell-5.1) | [Parsing and stop-parsing](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_parsing?view=powershell-5.1)
