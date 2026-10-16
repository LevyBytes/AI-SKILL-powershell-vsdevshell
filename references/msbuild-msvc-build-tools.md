# MSBuild and MSVC Build Tools

Open when invoking MSBuild or MSVC with an explicit configuration, architecture and verified Visual Studio toolchain.

## MSBuild invocation

Initialize the selected developer environment before build steps; use [Developer Shell setup](visual-studio-devshell.md). A fully configured build system may provide absolute tool paths and environment settings instead. The project determines valid configuration/platform names.

```powershell
$msbuild = (Get-Command msbuild.exe -CommandType Application -ErrorAction Stop | Select-Object -First 1).Source
$msbuildArgs = @(
    '<REPO_ROOT>\Project.sln',
    '-restore',
    '-m',
    '-t:Build',
    '-p:Configuration=Release',
    '-p:Platform=x64',
    '-verbosity:minimal',
    '-bl:<WORK_ROOT>\build.binlog'
)
& $msbuild @msbuildArgs
$exitCode = $LASTEXITCODE
if ($exitCode -ne 0) { throw ('MSBuild failed: {0}' -f $exitCode) }
```

MSBuild accepts case-insensitive `-switch` and `/switch` forms. Without `-m`, the default maximum process count is one. Keep property values and semicolon/comma lists in quoted argument elements so PowerShell passes each switch intact.

`-targets` (or `-ts`) lists available targets. MSBuild 17.8 introduced `-getProperty`, `-getItem` and `-getTargetResult`. Property/item queries without requested targets evaluate without a build; adding `-target` or `-getTargetResult` can execute targets. Do not describe every property query as side-effect-free: project evaluation can itself load extensions or evaluate property functions.

Since Visual Studio 2019 16.5, MSBuild/DEVENV toolset and library selection is controlled by the build configuration rather than simply inheriting a Developer Prompt's selection. The shell still affects executable discovery; direct `cl.exe` invocation depends on the configured environment.

## MSVC environment and real compiler check

Check the selected executable and expected architecture before compiling. `-Arch amd64` is an initialization argument; standard x64 developer environments report `VSCMD_ARG_TGT_ARCH` as `x64`. A defined `VSCMD_VER` alone does not establish the intended installation or architecture.

```powershell
$compiler = (Get-Command cl.exe -CommandType Application -ErrorAction Stop | Select-Object -First 1).Source
if ([string]::IsNullOrWhiteSpace($Env:INCLUDE) -or [string]::IsNullOrWhiteSpace($Env:LIB)) {
    throw 'INCLUDE/LIB are empty.'
}
if ($Env:VSCMD_ARG_TGT_ARCH -ne 'x64') { throw 'Expected an x64 target environment.' }
$sourcePath = Join-Path '<WORK_ROOT>' 'compiler-check.c'
$objectPath = Join-Path '<WORK_ROOT>' 'compiler-check.obj'
Set-Content -LiteralPath $sourcePath -Value 'int main(void) { return 0; }' -Encoding ASCII
& $compiler /nologo /c $sourcePath ('/Fo{0}' -f $objectPath)
$compileExit = $LASTEXITCODE
if ($compileExit -ne 0) { throw ('Compiler check failed: {0}' -f $compileExit) }
if (-not (Test-Path -LiteralPath $objectPath -PathType Leaf)) { throw 'Object file missing.' }
```

Use an existing dedicated work directory; the two `compiler-check` files belong to this fixture. This check compiles C to an object only. It does not test Windows SDK headers, linking, runtime execution, or a complete project. Add those checks when they matter to the task.

`cl /Bv` without a source file can emit version details and then fail with a missing-source diagnostic. That nonzero exit code alone does not establish a broken environment. Inspect executable version metadata or provide a real source file when using compiler diagnostic options.

## Reporting and recovery

Record the Visual Studio instance, resolved compiler/linker/MSBuild paths, `VSCMD_VER`, architecture variables, `VCToolsInstallDir`, `WindowsSdkDir`, arguments, exit codes and artifact paths. Keep logs in the assigned work area.

If standard headers or libraries are missing, inspect the selected toolset, SDK and configuration. Start a fresh correctly initialized shell when the environment is wrong. Do not mix installation command files or invent include/library paths to mask a selection problem.

## Primary sources

[MSBuild switches](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-command-line-reference?view=visualstudio) | [C++ command-line tools](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170)
