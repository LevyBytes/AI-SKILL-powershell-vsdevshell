# PowerShell 5.1 Pitfalls Checklist

Before editing or generating Windows PowerShell scripts, check these items.

## Syntax

- [ ] No ternary operator (`? :`).
- [ ] No null-coalescing (`??`) or null-coalescing assignment (`??=`).
- [ ] No null-conditional member access (`?.` / `?[]`).
- [ ] No pipeline chain operators (`&&` / `||`).
- [ ] No `ForEach-Object -Parallel`.
- [ ] No `Join-String` cmdlet (use the `-join` operator).
- [ ] No `` `e `` / `` `u{} `` escape sequences (PS 6+ only).
- [ ] Intended engine selected; `#Requires -Version 5.1` is a minimum, with `-PSEdition Desktop` when Windows PowerShell is required.
- [ ] Script parses cleanly under `powershell.exe` (AST ParseFile, errors listed with line numbers).
- [ ] No smart quotes or typographic punctuation in code-generated strings.

## Native commands

- [ ] Quoted executable paths invoked with `&`; executable path and arguments kept separate.
- [ ] Ordinary arguments built separately; empty strings, literal quotes and backslashes tested at the native boundary, or complex data passed through a file.
- [ ] No giant single-string command lines expecting `cmd.exe` splitting.
- [ ] `--%` stop-parsing token used (and its limits respected) when literal argument text is required.
- [ ] `$LASTEXITCODE` captured immediately after each native command.
- [ ] `$?` not read after grouping `(...)`/`$(...)`/`@(...)` (5.1 resets it to True).
- [ ] Stderr logged independently from exit status; direct process capture used when 5.1 redirection under Stop would raise NativeCommandError.
- [ ] Interactive blockers handled explicitly and documented.

## Visual Studio tools

- [ ] Developer environment loaded once at the top of build scripts, then verified.
- [ ] `VSCMD_VER` checked to detect an already-initialized shell.
- [ ] `-Arch`/`-HostArch` (or `-arch=`/`-host_arch=`) match the intended target/host; `-SkipAutomaticLocation` used in automation.
- [ ] `cl.exe`, `link.exe`, `msbuild.exe` resolved; `INCLUDE`/`LIB` non-empty; arch variables recorded.
- [ ] `vswhere.exe` located explicitly; discovery includes `-products '*'` when Build Tools must be considered.
- [ ] One Visual Studio instance chosen explicitly when several are installed; no mixed-version environments in one window.
- [ ] MSBuild called with explicit `-t:`/`-p:Configuration`/`-p:Platform`; `-m` for parallel; semicolon lists quoted from PowerShell.

## Files and generated scripts

- [ ] Artifacts routed to `<WORK_ROOT>`/`<REPORTS_ROOT>`/`<PROMPTS_ROOT>`; nothing written inside `.agents\skills`.
- [ ] Encoding passed explicitly on every write; 5.1 `-Encoding UTF8` BOM behavior accounted for (BOM-less .NET writer or `utf-8-sig` readers when needed).
- [ ] Long-path risk considered; artifact roots kept short.
- [ ] Generated Python written as single-quoted line arrays; values injected via argv/env/JSON, not interpolation; `py_compile` run.
- [ ] Generated JSON parses; JSONL parses line by line; cross-references (ids, files) validated.
- [ ] No hardlinks or symlinks unless explicitly authorized.

## Reporting

- [ ] Validation commands and exit codes recorded.
- [ ] Skipped checks stated with reasons.
- [ ] Git validation follows task scope; skips are attributed to the user only when actually requested.
- [ ] Reusable docs use placeholders instead of project-specific paths.
- [ ] Final status uses deterministic labels.
