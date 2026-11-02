---
name: windows-bash-setup
description: Fixes pi's bash tool failing with "No bash shell found" on Windows. Use when every bash command errors with "No bash shell found" (even `echo test`), or when setting up pi on a Windows machine where Git for Windows is installed in a non-default location.
---

# Windows Bash Setup

## Symptom

Every `bash` tool call fails instantly with:

```
No bash shell found. Options: ... Set shellPath in settings.json
Searched Git Bash in:
  C:\Program Files\Git\bin\bash.exe
  C:\Program Files (x86)\Git\bin\bash.exe
```

The shell is broken for **all** commands, regardless of what they are.

## Root Cause

On Windows, pi runs the `bash` tool through Git Bash. It searches for a shell in order:

1. `shellPath` from `~/.pi/agent/settings.json` (custom)
2. Default install paths `C:\Program Files\Git\bin\bash.exe` / `C:\Program Files (x86)\Git\bin\bash.exe`
3. `bash.exe` on PATH (Cygwin, MSYS2, WSL)

If Git for Windows is installed somewhere else (e.g. `E:\Git`, portable installs, scoop), none of these match and every command fails. Note: `settings.json` is read **at pi startup** — changes require restarting pi.

## Critical: Diagnose WITHOUT bash

Because bash is broken, do not attempt to use it. Use the `read` tool to probe candidate paths:

- **File exists** → `read` returns binary data (starts with `MZ`) or a "binary file" style dump
- **File missing** → `read` returns `ENOENT: no such file or directory`

## Config File Locations

| Config | Path |
|--------|------|
| Global settings | `~/.pi/agent/settings.json` (in this environment: `C:\Users\rijul\.pi\agent\settings.json`) |
| Project settings | `.pi/settings.json` in the project root |

## Apply the Fix

### 1. Find Git Bash

Probe these paths with `read` until one returns binary data (exists):

```
E:\Git\bin\bash.exe                              # known install on this machine
C:\Program Files\Git\bin\bash.exe
C:\Program Files (x86)\Git\bin\bash.exe
C:\msys64\usr\bin\bash.exe                       # MSYS2
C:\Users\<user>\scoop\apps\git\current\usr\bin\bash.exe
C:\Users\<user>\AppData\Local\Programs\Git\bin\bash.exe
```

### 2. Set `shellPath`

Add (or update) the `shellPath` key in the **global** settings file. Backslashes must be doubled as JSON requires:

```json
{
  "shellPath": "E:\\Git\\bin\\bash.exe"
}
```

Read the existing file first and merge the key in — do not clobber other settings.

### 3. Restart pi

The setting is read at startup. Ask the user to restart pi. Do **not** retry the bash tool expecting it to work before the restart — it will keep failing with the cached paths.

### 4. Verify after restart

```bash
echo OK && bash --version
```

## Fallbacks (if no Git Bash exists)

- **Install Git for Windows**: https://git-scm.com/download/win — needs the user to run the installer; remove `shellPath` afterwards so the default search-path finds it, or point `shellPath` at the new location.
- **Use PowerShell instead**: pi ships a `powershell` tool on Windows. Replace the model-facing `bash` tool in settings.json (requires restart):
  ```json
  {
    "defaultTools": ["read", "powershell", "edit", "write"]
  }
  ```
- **Git via DevTools / other tools**: `git.exe` can exist without `bash.exe` in some bundles (e.g. VS Code's bundled Git has no bash). Keep looking for a real MSYS2/MinGW bash rather than settling for a bare `git.exe`.

## References

- pi Windows setup docs: `docs/windows.md` (section "Custom Bash Path" and "PowerShell Tool")