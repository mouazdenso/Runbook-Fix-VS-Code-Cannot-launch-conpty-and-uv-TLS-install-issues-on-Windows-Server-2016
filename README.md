# Runbook-Fix-VS-Code-Cannot-launch-conpty-and-uv-TLS-install-issues-on-Windows-Server-2016



## Scope

This guide applies to older Windows servers like our environment: **Microsoft Hyper-V Server 2016 / Windows Server 2016, version 1607, build 14393.8957**.

## Symptoms

You may see one or both of these problems:

1. **VS Code integrated terminal fails**

   ```text
   The terminal process failed to launch: A native exception occurred during launch (Cannot launch conpty).
   ```

2. **`uv` install fails in PowerShell**

   ```text
   irm : The request was aborted: Could not create SSL/TLS secure channel.
   ```

## Root cause

Starting with **VS Code 1.109**, `winpty` support was removed. As a result, the integrated terminal no longer works on Windows versions before **Windows 10 / Server build 1809** unless the experimental ConPTY DLL setting happens to work.

On older Windows PowerShell environments, HTTPS downloads can also fail because the session is not using **TLS 1.2** by default.

## Resolution steps

### Step 1: Confirm the Windows build

Run:

```powershell
winver
```

If the machine is **version 1607 / build 14393**, this issue is expected with newer VS Code releases.

### Step 2: Try the VS Code workaround first

Open VS Code `settings.json` and add:

```json
"terminal.integrated.windowsUseConptyDll": true
```

Then fully close and reopen VS Code.

### Step 3: If terminal still fails, downgrade VS Code

If Step 2 does not work, uninstall the current VS Code and install **VS Code 1.108.2**.

After downgrade, disable auto-update in `settings.json`:

```json
"update.mode": "none"
```

### Step 4: Use external PowerShell if VS Code terminal is unavailable

Until VS Code is fixed, open **PowerShell outside VS Code** and run commands there.

### Step 5: Fix the `uv` TLS error

Before running the installer, force the session to use TLS 1.2:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Step 6: Verify `uv` installation

Run:

```powershell
uv --version
where.exe uv
```

Expected result:

```text
C:\Program Files\uv\uv.exe
```

If `where.exe uv` shows multiple paths, the **first path listed** is the one Windows is using.

### Step 7: Install Python with `uv`

To install Python through `uv`:

```powershell
uv python install 3.12 --default
python --version
```

### Step 8: If VS Code terminal is still unreliable, use a task

Create `.vscode\tasks.json` in the project and call `uv` directly:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "uv version",
      "type": "shell",
      "command": "C:\\Program Files\\uv\\uv.exe",
      "args": ["--version"],
      "problemMatcher": []
    }
  ]
}
```

## Validation checklist

Use this checklist after the fix:

* `winver` confirms whether the OS is older than 1809
* VS Code integrated terminal opens successfully
* `uv --version` returns a version
* `where.exe uv` shows `C:\Program Files\uv\uv.exe` first
* `python --version` works after `uv python install 3.12 --default`

## Recommended final state

For this server class, the cleanest setup is:

* VS Code **1.108.2** on Windows Server 2016 / build 14393
* `uv` installed and resolving from `C:\Program Files\uv\uv.exe`
* Python installed through `uv`
* Auto-update disabled until the OS is upgraded

## Long-term recommendation

The real long-term fix is to upgrade the server OS to a Windows release that supports the modern ConPTY path required by newer VS Code versions.

