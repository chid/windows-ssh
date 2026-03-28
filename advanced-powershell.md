# Advanced PowerShell Tips

This guide focuses on higher-leverage PowerShell improvements for people who spend a lot of time managing Windows over SSH. The goal is to make your shell faster to use, safer for admin work, and easier to turn into a personal toolkit.

## Build A Real Profile

Treat your PowerShell profile like a small configuration file for your working environment, not just a place for one-off aliases.

Profile path:

```powershell
$PROFILE
```

Create it if needed:

```powershell
if (-not (Test-Path $PROFILE)) {
    New-Item -ItemType File -Path $PROFILE -Force
}
```

Useful baseline:

```powershell
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

Set-Alias ll Get-ChildItem
Set-Alias grep rg

function la { Get-ChildItem -Force @args }
function .. { Set-Location .. }
function which($Name) { Get-Command $Name | Select-Object Name, Source }
function reload-profile { . $PROFILE }

function df {
    Get-Volume |
        Select-Object DriveLetter, FileSystemLabel,
        @{N='SizeGB';E={[math]::Round($_.Size / 1GB, 2)}},
        @{N='FreeGB';E={[math]::Round($_.SizeRemaining / 1GB, 2)}}
}
```

Why this helps:

- safer defaults for scripts
- fewer repetitive commands
- easier debugging
- faster reconnect recovery over SSH

## Upgrade The Interactive Experience

`PSReadLine` is one of the highest-payoff PowerShell upgrades because it improves line editing, completion, and history.

Useful settings:

```powershell
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
Set-PSReadLineKeyHandler -Key Ctrl+r -Function ReverseSearchHistory
Set-PSReadLineOption -EditMode Windows
```

What you get:

- shell history suggestions as you type
- much better command recall
- easier tab completion for long commands
- less friction in remote sessions

## Prefer Objects Over Text Parsing

PowerShell gets much more useful when you stop treating it like `cmd.exe`.

Instead of:

```powershell
ipconfig | findstr IPv4
```

Prefer:

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
    Select-Object InterfaceAlias, IPAddress
```

Instead of:

```powershell
tasklist
```

Prefer:

```powershell
Get-Process |
    Sort-Object CPU -Descending |
    Select-Object -First 20 Name, Id, CPU
```

Object-first commands are easier to filter, sort, export, and reuse in functions.

## Write Advanced Functions Instead Of Throwaway Commands

If you repeat a command more than a few times, turn it into a proper function.

Good PowerShell functions should often use:

- `[CmdletBinding()]`
- named parameters
- validation
- pipeline support when helpful
- `SupportsShouldProcess` for potentially destructive actions

Example:

```powershell
function Clear-TempSafe {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [string[]]$Path = @($env:TEMP, 'C:\Windows\Temp')
    )

    foreach ($p in $Path) {
        if (Test-Path $p) {
            if ($PSCmdlet.ShouldProcess($p, 'Remove temp files')) {
                Get-ChildItem $p -Force -ErrorAction SilentlyContinue |
                    Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
            }
        }
    }
}
```

Usage:

```powershell
Clear-TempSafe -WhatIf
Clear-TempSafe
```

That pattern is much safer than pasting destructive one-liners into a live SSH session.

## Create A Personal Module

Once your profile starts getting crowded, move reusable functions into a module.

Suggested path:

```text
$HOME\Documents\PowerShell\Modules\CharleyTools\CharleyTools.psm1
```

Good module candidates:

- `Get-LargeDirectory`
- `Clear-TempSafe`
- `Find-PortOwner`
- `Get-DiskReport`
- `Restart-ServiceSafe`
- `Get-RecentErrors`
- `Edit-Profile`
- `Open-SSHConfig`

Benefits:

- cleaner profile
- easier reuse across machines
- easier versioning
- simpler testing

## Add SSH-Focused Helper Functions

Remote admin sessions benefit from a few specific helpers.

### Find what owns a port

```powershell
function Find-PortOwner {
    param([int]$Port)

    Get-NetTCPConnection -LocalPort $Port -ErrorAction SilentlyContinue |
        Select-Object LocalAddress, LocalPort, State, OwningProcess,
        @{N='ProcessName';E={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}}
}
```

### Show recent important errors

```powershell
function Get-RecentErrors {
    param([int]$Hours = 4)

    $start = (Get-Date).AddHours(-$Hours)
    Get-WinEvent -FilterHashtable @{
        LogName = 'System'
        Level = 1,2,3
        StartTime = $start
    } | Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message
}
```

### Quick disk report

```powershell
function Get-DiskReport {
    Get-Volume |
        Sort-Object DriveLetter |
        Select-Object DriveLetter, FileSystemLabel,
        @{N='SizeGB';E={[math]::Round($_.Size / 1GB, 2)}},
        @{N='FreeGB';E={[math]::Round($_.SizeRemaining / 1GB, 2)}},
        @{N='UsedPct';E={
            if ($_.Size -gt 0) {
                [math]::Round((1 - ($_.SizeRemaining / $_.Size)) * 100, 1)
            }
        }}
}
```

## Use `-WhatIf` And `try/catch` Aggressively

PowerShell is excellent for admin work, which also means it is easy to do real damage quickly.

Strong habits:

- use `SupportsShouldProcess`
- test destructive functions with `-WhatIf`
- prefer `try/catch` over silent failure
- set `$ErrorActionPreference = 'Stop'` in automation

Pattern:

```powershell
try {
    Restart-Service -Name sshd -ErrorAction Stop
}
catch {
    Write-Error "Failed to restart sshd: $($_.Exception.Message)"
}
```

## Parallelize Slow Work

PowerShell 7 adds useful parallel options.

Example:

```powershell
$paths = 'C:\Users', 'C:\ProgramData', 'C:\Windows\Temp'

$paths | ForEach-Object -Parallel {
    Get-ChildItem $_ -Force -ErrorAction SilentlyContinue |
        Select-Object FullName
} -ThrottleLimit 4
```

Good uses:

- scanning multiple folders
- collecting logs from several targets
- running repeated diagnostics
- gathering inventory data

For simpler background work:

```powershell
Start-Job { Get-Date; Start-Sleep 30; 'done' }
Get-Job
Receive-Job -Wait -AutoRemoveJob
```

## Use Better Output And Reports

PowerShell is strong at turning raw data into something readable.

Example:

```powershell
Get-Process |
    Sort-Object WorkingSet -Descending |
    Select-Object -First 15 Name, Id,
    @{N='RAM_MB';E={[math]::Round($_.WorkingSet / 1MB, 1)}} |
    Format-Table -AutoSize
```

And when you need structured output:

```powershell
Get-Service | Export-Csv .\services.csv -NoTypeInformation
Get-ComputerInfo | ConvertTo-Json -Depth 3
```

## Add Better Navigation And Search

PowerShell gets much nicer when you combine it with a few good external tools:

- `rg` for search
- `fd` for finding files
- `gdu` for disk usage
- `jq` for JSON
- `fzf` for fuzzy selection
- `zoxide` for faster directory jumping

These tools complement PowerShell well instead of replacing it.

## Consider Prompt And Completion Upgrades

If you spend hours a day in the shell, polish matters.

Useful additions:

- `oh-my-posh` for a better prompt
- `Terminal-Icons` for richer file listings
- `fzf` history and path selection
- custom tab completion for your own functions

This is not strictly necessary, but it can make a frequently used admin shell feel much better.

## Good Advanced Workflow

A practical mature setup often looks like this:

1. Use PowerShell 7 instead of legacy Windows PowerShell where possible.
2. Keep a clean `$PROFILE` for shell startup and ergonomics.
3. Move reusable functions into a personal module.
4. Use object-based commands for inspection and reporting.
5. Add `-WhatIf` and error handling to anything destructive.
6. Use jobs or parallel execution for slow repetitive work.
7. Pair PowerShell with a few high-quality CLI tools.

## Suggested Starter Build

If you want a strong next step, build these first:

1. A polished profile with aliases, `PSReadLine`, and a few helper functions.
2. A `CharleyTools` module with 5 to 10 admin-focused commands.
3. A small set of reporting commands for disks, services, ports, and event logs.
4. A safe cleanup toolkit that defaults to `-WhatIf` patterns.

That gives you a setup that is much more pleasant over SSH and much easier to extend over time.
