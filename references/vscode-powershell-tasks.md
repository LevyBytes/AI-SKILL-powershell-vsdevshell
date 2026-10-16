# VS Code PowerShell sessions and tasks

Open when selecting the PowerShell engine in VS Code or diagnosing task arguments and working directories. Follow the caller's repository instructions for paths, permissions and artifacts.

## Select the engine

With Microsoft's PowerShell extension installed, run **PowerShell: Show Session Menu** and select the installed engine required by the project. Verify `$PSVersionTable` in the extension terminal. Windows PowerShell 5.1 support is best effort; the extension requires .NET Framework 4.8 or later for that engine. The extension terminal and a task's shell are separate choices.

## Configure a task

In `.vscode/tasks.json`, `type: "process"` launches the executable directly; `type: "shell"` passes the command through a shell. Use the task's `args` array and explicit working directory instead of assuming the open editor determines either. A minimal process task for an existing script is:

```json
{
  "version": "2.0.0",
  "tasks": [{
    "label": "Validate Windows script",
    "type": "process",
    "command": "${env:WINDIR}\\System32\\WindowsPowerShell\\v1.0\\powershell.exe",
    "args": ["-NoProfile", "-NonInteractive", "-File", "${workspaceFolder}/scripts/validate.ps1"],
    "options": {"cwd": "${workspaceFolder}"},
    "problemMatcher": []
  }]
}
```

Substitute the actual script path. For debugging, select the required extension session before starting the PowerShell debug configuration. A task exit code and a debugger session verify different behavior.

Sources: [PowerShell in VS Code](https://code.visualstudio.com/docs/languages/powershell) and [VS Code tasks](https://code.visualstudio.com/docs/debugtest/tasks).
