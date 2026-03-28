# Windows SSH QOL Quickstart

Make a Windows machine pleasant to SSH into.

This quickstart upgrades the default Windows SSH experience with a better shell, better tools, a real editor, and SSH key auth so you can stop typing passwords.

## Best improvements

- Use PowerShell or `pwsh` as the default shell instead of `cmd`.
- Install a better disk and file toolkit: `gdu`, `ripgrep`, `fd`, `jq`, `bat`, `git`.
- Add a real terminal editor: `vim`, `neovim`, or `nano`.
- Install a package manager so future setup is easy: `winget`, `scoop`, or `choco`.
- Enable OpenSSH key auth so you stop typing passwords.
- Add Windows Terminal for local use, even if the host is Windows.

## What this is for

Use this if:

- you SSH into a Windows box regularly
- you want sane CLI defaults instead of stock `cmd`
- you want Linux-style tools that work well over SSH
- you want a fast setup path with `winget`

This guide covers both:

- the Windows machine you will SSH into
- the local machine you will SSH from

## Prerequisites

Assumptions:

- the Windows host is a modern Windows 10 or Windows 11 machine
- you can open PowerShell as Administrator on the host when needed
- `winget` is available on the Windows host
- OpenSSH Server is available as a Windows feature

Open PowerShell on the Windows host and use that for the commands below.

## 1. Install the quality-of-life tools

Install the basics first:

```powershell
winget install Microsoft.PowerShell
winget install BurntSushi.ripgrep.MSVC
winget install sharkdp.fd
winget install jqlang.jq
winget install ducaale.gdu
winget install sharkdp.bat
winget install Neovim.Neovim
winget install Git.Git
```

Optional local-terminal upgrade:

```powershell
winget install Microsoft.WindowsTerminal
```

What each tool helps with:

- `pwsh`: a much better shell than `cmd`
- `rg`: fast text search across directories
- `fd`: better file finder
- `jq`: JSON formatting and querying
- `gdu`: quick disk usage analysis
- `bat`: `cat` with syntax highlighting and paging
- `nvim`: a real terminal editor
- `git`: required for modern CLI workflows

If you prefer a different package manager, `scoop` and `choco` also work, but this guide keeps `winget` as the main path.

## 2. Make PowerShell the default SSH shell

Windows OpenSSH often drops users into `cmd.exe` by default. Set PowerShell as the default shell so SSH sessions start somewhere usable.

Run this in an elevated PowerShell window on the Windows host:

```powershell
New-ItemProperty `
  -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Program Files\PowerShell\7\pwsh.exe" `
  -PropertyType String `
  -Force
```

If you want Windows PowerShell 5.1 instead of PowerShell 7:

```powershell
New-ItemProperty `
  -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
  -PropertyType String `
  -Force
```

Restart the SSH service after changing the shell:

```powershell
Restart-Service sshd
```

## 3. Install a real editor

For a simple modern default, use Neovim:

```powershell
winget install Neovim.Neovim
```

Then you can edit files over SSH with:

```powershell
nvim $PROFILE
```

If you prefer something simpler:

- `vim` is familiar to many Unix users
- `nano` is easier for quick edits

The important part is having any terminal editor at all so you are not stuck with awkward one-off file editing.

## 4. Enable OpenSSH Server on the Windows host

Check whether OpenSSH Server is installed:

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
```

If it is not installed:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Start the service and enable it on boot:

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

Make sure the firewall rule exists:

```powershell
Get-NetFirewallRule -Name *OpenSSH* | Select-Object Name, Enabled, Direction, Action
```

If needed, create the inbound rule:

```powershell
New-NetFirewallRule -Name sshd `
  -DisplayName "OpenSSH Server (sshd)" `
  -Enabled True `
  -Direction Inbound `
  -Protocol TCP `
  -Action Allow `
  -LocalPort 22
```

## 5. Switch to SSH keys

This is the highest-value improvement. Once key auth works, you stop typing your password on every login.

### On the local machine

Generate a key if you do not already have one:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

This usually creates:

- macOS/Linux: `~/.ssh/id_ed25519`
- Windows: `C:\Users\YOUR_USER\.ssh\id_ed25519`

### Copy the public key to the Windows host

Print your public key locally:

```bash
cat ~/.ssh/id_ed25519.pub
```

Then on the Windows host, create the `.ssh` directory if needed:

```powershell
New-Item -ItemType Directory -Force $HOME\.ssh
```

Append your public key into `authorized_keys`:

```powershell
notepad $HOME\.ssh\authorized_keys
```

Paste the full contents of your local `id_ed25519.pub` into that file, save, and close.

Lock down permissions enough for OpenSSH to accept the file:

```powershell
icacls $HOME\.ssh /inheritance:r
icacls $HOME\.ssh /grant "$($env:USERNAME):(OI)(CI)F"
icacls $HOME\.ssh\authorized_keys /inheritance:r
icacls $HOME\.ssh\authorized_keys /grant "$($env:USERNAME):F"
```

Restart SSH once after key setup:

```powershell
Restart-Service sshd
```

### Test key login

From your local machine:

```bash
ssh your-user@your-windows-host
```

If you use a non-default key:

```bash
ssh -i ~/.ssh/id_ed25519 your-user@your-windows-host
```

## 6. Client-side examples

### From macOS or Linux

Basic connection:

```bash
ssh your-user@host-or-ip
```

Add a host alias in `~/.ssh/config`:

```sshconfig
Host winbox
    HostName 192.168.1.50
    User your-user
    IdentityFile ~/.ssh/id_ed25519
```

Then connect with:

```bash
ssh winbox
```

### From Windows

Install Windows Terminal for local use if you do not already have it:

```powershell
winget install Microsoft.WindowsTerminal
```

Then connect from PowerShell:

```powershell
ssh your-user@host-or-ip
```

Or define a host in `C:\Users\YOUR_USER\.ssh\config`:

```sshconfig
Host winbox
    HostName 192.168.1.50
    User your-user
    IdentityFile C:\Users\YOUR_USER\.ssh\id_ed25519
```

Then use:

```powershell
ssh winbox
```

## 7. Recommended next steps

Once the basics are in place, the Windows host becomes much nicer to live in over SSH.

A good next wave of upgrades is:

- add aliases and functions to your PowerShell profile
- configure `git` with your name, email, and preferred editor
- use `rg`, `fd`, and `bat` as your default search and file-inspection tools
- use `gdu` when disk usage gets weird
- keep a small `~/.ssh/config` on your local machine so hostnames are memorable

## Quick command recap

```powershell
winget install Microsoft.PowerShell
winget install BurntSushi.ripgrep.MSVC
winget install sharkdp.fd
winget install jqlang.jq
winget install ducaale.gdu
winget install sharkdp.bat
winget install Neovim.Neovim
winget install Git.Git
winget install Microsoft.WindowsTerminal
```

```powershell
New-ItemProperty `
  -Path "HKLM:\SOFTWARE\OpenSSH" `
  -Name DefaultShell `
  -Value "C:\Program Files\PowerShell\7\pwsh.exe" `
  -PropertyType String `
  -Force

Restart-Service sshd
```

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
ssh your-user@host-or-ip
```
