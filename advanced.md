# Advanced Windows SSH Tips

This note covers higher-leverage quality-of-life improvements for working on Windows over SSH. The focus is on making remote sessions more reliable, more Unix-like, and less repetitive.

## Persistent Sessions

Windows SSH sessions do not have a native equivalent to `tmux` built in, so long-running work can be awkward if your connection drops.

Options:

- Use `tmux` inside WSL if your work can run there.
- Use `screen` inside WSL for a simpler persistent shell.
- Use PowerShell jobs for background tasks when WSL is not involved.
- Use Task Scheduler for unattended recurring or long-lived work.

Examples:

```powershell
Start-Job -Name cleanup -ScriptBlock { Get-Date; Start-Sleep 300; "done" }
Get-Job
Receive-Job -Name cleanup
```

For anything interactive or long-running, WSL plus `tmux` is usually the best experience.

## WSL As Your Unix Layer

If you spend a lot of time over SSH, WSL is one of the biggest upgrades you can make. It gives you a Linux shell, package manager, `tmux`, and the normal Unix tooling ecosystem without replacing Windows.

Install and check status:

```powershell
wsl --install
wsl --status
wsl -l -v
```

Why it helps:

- Better shell tooling
- Easier package installs
- Reliable support for `tmux`, `rsync`, `ssh-agent`, `htop`, and friends
- A much closer match to Linux server workflows

Good pattern:

- Keep Windows-native apps on Windows.
- Use WSL for shell-heavy workflows, scripting, and persistent terminal sessions.
- Avoid constantly crossing between `/mnt/c/...` and the Linux filesystem for heavy file operations because it can be slower.

## Better Shell Defaults

Set PowerShell as the default shell for OpenSSH instead of `cmd.exe`. It makes scripting, tab completion, and object-based commands much nicer.

Useful additions:

- `Microsoft.PowerShell` or `pwsh`
- `PSReadLine` for history and editing improvements
- a custom PowerShell profile with aliases and helper functions

Helpful aliases:

```powershell
Set-Alias ll Get-ChildItem
Set-Alias grep rg
function la { Get-ChildItem -Force }
function .. { Set-Location .. }
function which($name) { Get-Command $name }
```

Profile location:

```powershell
$PROFILE
```

## Better Tools

Install a few command-line tools that reduce friction immediately:

- `gdu` for disk usage
- `ripgrep` (`rg`) for fast search
- `fd` for finding files
- `jq` for JSON
- `bat` for readable file previews
- `git` for sane CLI workflows
- `7zip` for archives
- `neovim` or `vim` for editing

With `winget`:

```powershell
winget install Microsoft.PowerShell
winget install ducaale.gdu
winget install BurntSushi.ripgrep.MSVC
winget install sharkdp.fd
winget install jqlang.jq
winget install Git.Git
winget install Neovim.Neovim
```

## Use SSH Keys Everywhere

Password login gets old fast. Use SSH keys for the host itself and for Git remotes when possible.

Basic flow:

```powershell
ssh-keygen -t ed25519
Get-Service ssh-agent
ssh-add $HOME\.ssh\id_ed25519
```

Benefits:

- Faster logins
- Easier automation
- Better security than repeated password entry

## Improve Long-Running Tasks

A remote Windows machine often gets used for builds, downloads, indexing, model runs, or maintenance. These jobs are easier to manage if you separate execution from your interactive login session.

Options by use case:

- Short background work: PowerShell jobs
- Scheduled or resilient automation: Task Scheduler
- Interactive persistent terminals: WSL plus `tmux`
- Service-style workloads: NSSM, scheduled tasks, or a proper Windows service

If reconnect tolerance matters, do not rely on a single foreground SSH shell.

## Watch Out For Admin Boundaries

Remote Windows work often fails because the SSH session is not elevated. Some cleanup or service commands need admin rights.

Common examples:

- stopping services
- editing system-wide PATH
- changing OpenSSH server settings
- cleaning Windows Update caches

Consider installing `gsudo` if you frequently need elevation from a shell.

## Storage And Cache Strategy

If `C:` keeps filling up, it is often better to move heavy caches than to repeatedly clean them.

Candidates:

- package caches
- Docker data
- build artifacts
- model files
- VM images
- browser caches
- temp extraction directories

Tactics:

- move caches to a larger secondary drive
- use junctions or symlinks carefully
- keep large datasets out of user profile folders

## Make Reconnects Cheaper

A lot of SSH frustration is not the connection itself, but rebuilding context after reconnecting.

A few habits help:

- keep a shell profile with aliases and helper functions
- use persistent sessions where possible
- keep logs in files, not only in console output
- put recurring commands in scripts instead of ad hoc history
- standardize a few directories for downloads, temp work, and archives

## Good Default Setup

If you only do a few upgrades, these usually have the highest payoff:

1. Set PowerShell as the default SSH shell.
2. Install WSL and use `tmux` inside it.
3. Install `gdu`, `rg`, `fd`, `jq`, `git`, and `neovim`.
4. Switch to SSH keys.
5. Move heavy caches and temp data off `C:` when possible.
