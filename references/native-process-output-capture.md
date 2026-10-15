# Native Process Output Capture

Open when a Windows PowerShell 5.1 wrapper must capture stdout and stderr without turning a successful tool's diagnostic output into a terminating PowerShell error.

## Why use a process object

Under `$ErrorActionPreference = 'Stop'`, Windows PowerShell 5.1 can raise `NativeCommandError` for redirected stderr even if the program exits zero. Sending stderr to a file with `2>` does not by itself remove this behavior. A `System.Diagnostics.Process` instance reads the process streams directly and exposes its exit code independently.

## Complete Python job wrapper

Provide an existing approved Python executable, `job.py`, and `job-config.json`; the Python script reads its config path from `sys.argv[1]`. Replace the placeholders with absolute paths and ensure the work directory exists. This example's arguments are file paths, which cannot contain literal double quotes on Windows and do not end in a directory separator. Put arbitrary text values in the JSON file instead of embedding them in the command string.

```powershell
$python = '<TOOLS_ROOT>\python.exe'
$workRoot = '<WORK_ROOT>'
$jobPath = Join-Path $workRoot 'job.py'
$configPath = Join-Path $workRoot 'job-config.json'
foreach ($path in @($python, $jobPath, $configPath)) {
    if (-not (Test-Path -LiteralPath $path -PathType Leaf)) {
        throw ('Missing input file: {0}' -f $path)
    }
}
$utf8 = New-Object System.Text.UTF8Encoding($false)
$start = New-Object System.Diagnostics.ProcessStartInfo
$start.FileName = $python
$start.Arguments = '"{0}" "{1}"' -f $jobPath, $configPath
$start.WorkingDirectory = $workRoot
$start.UseShellExecute = $false
$start.CreateNoWindow = $true
$start.RedirectStandardOutput = $true
$start.RedirectStandardError = $true
$start.StandardOutputEncoding = $utf8
$start.StandardErrorEncoding = $utf8
$start.EnvironmentVariables['PYTHONIOENCODING'] = 'utf-8'
$process = New-Object System.Diagnostics.Process
$process.StartInfo = $start
try {
    if (-not $process.Start()) { throw 'Process did not start.' }
    $stdoutRead = $process.StandardOutput.ReadToEndAsync()
    $stderrRead = $process.StandardError.ReadToEndAsync()
    $process.WaitForExit()
    $stdout = $stdoutRead.GetAwaiter().GetResult()
    $stderr = $stderrRead.GetAwaiter().GetResult()
    $exitCode = $process.ExitCode
} finally {
    $process.Dispose()
}
[System.IO.File]::WriteAllText((Join-Path $workRoot 'job.stdout.log'), $stdout, $utf8)
[System.IO.File]::WriteAllText((Join-Path $workRoot 'job.stderr.log'), $stderr, $utf8)
if ($exitCode -ne 0) { throw ('Python job failed: {0}' -f $exitCode) }
```

Start both reads before waiting so either stream can drain while the child runs. This wrapper waits for completion and holds both outputs in memory; use bounded streaming and a timeout/cancellation policy for long-running or untrusted jobs. Disposal releases the process handle; it is not a command to terminate a running process.

The explicit UTF-8 contract applies to this Python child. Other tools require their documented encoding and argument-quoting rules. `Start-Process -ArgumentList` and .NET Framework's `ProcessStartInfo.Arguments` still serialize arguments into a string; neither is a universal fix for arbitrary argument values.

## Related

[Native quoting](quoting-and-native-commands.md) · [Errors and logging](errors-exit-codes-and-logging.md) · [Generated Python](embedded-python-in-powershell.md) · [Encoding](files-paths-and-encoding.md)
