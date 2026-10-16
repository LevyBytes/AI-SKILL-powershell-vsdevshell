# Visual Studio Developer Shell

Visual Studio command-line tools require a developer environment. Loading must be verified, not assumed.

## Developer PowerShell (Launch-VsDevShell.ps1)

`Launch-VsDevShell.ps1` lives in `<VSINSTALLDIR>\Common7\Tools\`. It locates the Microsoft.VisualStudio.DevShell.dll module in that installation and invokes `Enter-VsDevShell`, updating the CURRENT PowerShell process environment. It is the recommended way to initialize Developer PowerShell for scripting.

```powershell
$launch = '<VSINSTALLDIR>\Common7\Tools\Launch-VsDevShell.ps1'
& $launch -Arch amd64 -HostArch amd64 -SkipAutomaticLocation
```

- `-Arch <target>` / `-HostArch <host>`: target/host architecture (`x86` default, `amd64`, `arm`, `arm64` as target). Available since Visual Studio 2022 17.1. Setting only `-Arch` makes the shell try to match the host - set both for cross builds.
- `-SkipAutomaticLocation`: keeps the shell in the current directory instead of jumping to the Visual Studio project location - almost always what an automation script wants.
- Execution policy must allow the script to run.

## Developer Command Prompt (VsDevCmd.bat / vcvarsall.bat)

`VsDevCmd.bat` is in `<VSINSTALLDIR>\Common7\Tools\`; architecture-specific `vcvars*.bat`/`vcvarsall.bat` are in `<VSINSTALLDIR>\VC\Auxiliary\Build\`. Their architecture arguments differ: `VsDevCmd.bat -arch=amd64 -host_arch=amd64` selects an x64 host and target; `vcvarsall.bat amd64` selects that native toolchain, while `vcvarsall.bat amd64_arm64` selects an x64 host targeting ARM64. Do not pass VsDevCmd flags to vcvarsall.

```powershell
$vsdevcmd = '<VSINSTALLDIR>\Common7\Tools\VsDevCmd.bat'
& cmd.exe /c ('call "{0}" -arch=amd64 -host_arch=amd64 && set' -f $vsdevcmd)
```

Environment changes inside a child `cmd.exe` DO NOT update the parent PowerShell process. If the current PowerShell process needs the environment, use `Launch-VsDevShell.ps1`, or parse the `set` output and import the variables explicitly. Cautions: do not mix command files from different Visual Studio versions in one window, and do not switch environments in the same window - start a fresh shell instead.

## Discovery without assuming paths

- `<VSINSTALLDIR>` varies by version, edition and chosen installation directory. Editions and Build Tools can use different Program Files roots. Use the actual `installationPath` from discovery rather than constructing it from a version and product name.
- `vswhere.exe` need not be on PATH; its documented install location is `%ProgramFiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe` (included with VS 2017+; also a separate download). Example listing all instances: `& $vswhere -products '*' -legacy -prerelease -format json`, where `$vswhere` is the resolved executable path. `-products '*'` includes Build Tools; without it, the default product selection omits that SKU.
- Build Tools SKUs and components are identified by IDs such as `Microsoft.Component.MSBuild` and the `Microsoft.VisualStudio.Workload.VCTools` workload. vswhere supports filtering by product and required components - see `vswhere -?` for `-products` and `-requires`.
- Multiple instances/editions can coexist on one machine. Pick one instance explicitly and record which one was used; do not let PATH luck decide.
- If the repo's instructions provide resolved paths, prefer those over discovery.

## Detecting an already-loaded environment

Since Visual Studio 2015, the developer shells set `VSCMD_VER`. The recommended way to detect whether a developer environment is already initialized in the current console is to test whether `VSCMD_VER` is defined:

```powershell
$developerEnvironmentLoaded = -not [string]::IsNullOrWhiteSpace($Env:VSCMD_VER)
```

`VSCMD_VER` is an initialization marker, not proof of the selected installation, SDK or architecture. Compare the variables below with the intended build before reusing an existing shell.

## Verification checklist (after loading)

Verify and record in the task log - loading is evidence to verify, not a state to assume:

- `VSCMD_VER`, `VisualStudioVersion`
- `VSINSTALLDIR`, `VCINSTALLDIR`, `VCToolsInstallDir`, `WindowsSdkDir`
- `VSCMD_ARG_TGT_ARCH`, `VSCMD_ARG_HOST_ARCH` (match the intended target/host)
- `cl.exe`, `link.exe`, `msbuild.exe` resolvable (`Get-Command cl.exe`)
- `INCLUDE` and `LIB` non-empty and pointing at the expected VS + Windows SDK roots
- `PATH` contains the toolset directories

Treat missing variables or tools as a setup blocker, not as a reason to guess paths. Example values (such as `VSCMD_VER 17.x` with a BuildTools install root) are LOCAL evidence from one machine, never universal assumptions.

Sources: [Developer shells](https://learn.microsoft.com/en-us/visualstudio/ide/reference/command-prompt-powershell?view=visualstudio) and [vcvarsall syntax and optional SDK/toolset arguments](https://learn.microsoft.com/en-us/cpp/build/building-on-the-command-line?view=msvc-170#vcvarsall-syntax).
