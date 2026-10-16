# Arrays, Hashtables, and Functions

Use simple Windows PowerShell 5.1 constructs for portable scripts.

## Arrays

`@(...)` always produces an array of zero or more objects; `+=` on an array creates a new array each time, which is slow in loops:

```powershell
$items = @()
$items += 'one'        # fine for a handful of items
```

For accumulation in loops, prefer a generic list:

```powershell
$items = New-Object 'System.Collections.Generic.List[string]'
[void]$items.Add('one')
[void]$items.Add('two')
```

Pipelines enumerate arrays. Preserve an array as ONE object with the unary comma:

```powershell
return ,$items
```

Joining and splitting use the 5.1 operators, not the PS7 `Join-String` cmdlet:

```powershell
$line  = $items -join ', '          # binary -join with delimiter
$parts = 'a;b;c' -split ';'         # -split delimiter is a REGEX by default; escape or use [regex]::Escape
```

Precedence gotcha: unary `-join "a","b"` joins only `"a"`; parenthesize: `-join ("a","b")`.

Comparison operators FILTER arrays instead of returning a Boolean when the left side is an array. That is why null checks put `$null` on the left:

```powershell
if ($null -eq $value) { Write-Output 'No value' } # $value is supplied by the caller
```

## Hashtables

Hashtable key order is not deterministic; use `[ordered]` when order matters:

```powershell
$record = [ordered]@{
    status = 'VALIDATED'
    path   = '<WORK_ROOT>\out.json'
}
```

`[ordered]` is only valid immediately before a hash literal (`$h = [ordered]@{...}`), never before the variable name. Iterate with `GetEnumerator()` when you need key-value pairs; sort for display with `GetEnumerator() | Sort-Object Key`.

## Functions

Prefer explicit parameters, `[CmdletBinding()]` for common-parameter support, and fail-early validation:

```powershell
function Write-TaskLog {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)][string]$Path,
        [Parameter(Mandatory=$true)][string]$Message
    )
    Add-Content -LiteralPath $Path -Value $Message -Encoding UTF8
}
```

- Define functions before they are called in scripts.
- Every uncaptured expression in a function is output; `return` exits but does not suppress earlier output.
- Use `-LiteralPath` for paths that may contain wildcard characters.
- Verb-Noun names with approved verbs for shared functions.
- Avoid hidden dependence on global variables unless the script intentionally sets shared task state near the top.

## Primary sources

[Arrays](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_arrays?view=powershell-5.1) | [Hashtables](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_hash_tables?view=powershell-5.1) | [Functions](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions?view=powershell-5.1)
