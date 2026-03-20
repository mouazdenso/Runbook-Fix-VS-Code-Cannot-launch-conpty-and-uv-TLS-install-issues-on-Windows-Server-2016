# Runbook: Fix VS Code `Cannot launch conpty` and `uv` TLS install issues on Windows Server 2016

## Purpose

This runbook explains how to diagnose and fix two common issues seen on older Windows Server environments:

1. VS Code integrated terminal fails with:

   ```text
   The terminal process failed to launch: A native exception occurred during launch (Cannot launch conpty)
   ```
2. `uv` installation fails in PowerShell with:

   ```text
   irm : The request was aborted: Could not create SSL/TLS secure channel.
   ```

This document is written as a step-by-step support guide so another engineer can reproduce the fix without needing prior context.

---

## Environment where this issue was observed

* OS: Microsoft Hyper-V Server 2016 / Windows Server 2016
* Version: 1607
* OS Build: 14393.x
* VS Code: Newer release line where integrated terminal behavior changed
* Shell: Windows PowerShell
* Package/runtime tool: `uv`

---

## Problem summary

Two separate issues appeared in the same environment:

### Issue A: VS Code integrated terminal does not open

Symptom:

```text
The terminal process failed to launch: A native exception occurred during launch (Cannot launch conpty)
```

Impact:

* The integrated terminal inside VS Code cannot be used
* Tasks and terminal-driven workflows may fail or behave inconsistently
* Engineers may incorrectly assume PowerShell or `uv` is broken, when the actual problem is VS Code terminal compatibility

### Issue B: `uv` installer fails with TLS/SSL error

Symptom:

```text
irm : The request was aborted: Could not create SSL/TLS secure channel.
```

Impact:

* Downloading the `uv` install script from PowerShell fails
* The failure happens before the installer runs
* This is usually caused by the PowerShell session not using TLS 1.2 by default on older Windows environments

---

## Root cause analysis

### Root cause for Issue A

The server is running an older Windows build: **1607 / 14393**.

This build is too old for the newer VS Code integrated terminal behavior after `winpty` support was removed. As a result:

* newer VS Code versions may fail to launch the integrated terminal
* the experimental ConPTY DLL workaround may or may not help
* the most reliable workaround on this OS is often to use an older VS Code version that still works on this platform

### Root cause for Issue B

The server’s PowerShell session does not always negotiate a modern TLS channel automatically.

When running:

```powershell
irm https://astral.sh/uv/install.ps1 | iex
```

PowerShell tries to establish HTTPS but fails because TLS 1.2 is not set for that session.

This is not a `uv` bug. It is a session-level TLS compatibility problem on the older Windows environment.

---

## High-level resolution strategy

Use this order:

1. Confirm OS version
2. Try the VS Code workaround setting
3. If still broken, downgrade VS Code
4. Use external PowerShell for installation tasks
5. Force TLS 1.2 in PowerShell
6. Install `uv`
7. Verify `uv`
8. Install Python through `uv`
9. Create a VS Code task as a stable fallback

---

## Step-by-step resolution

## Step 1: Confirm the Windows version

Open PowerShell and run:

```powershell
winver
```

Or simply type:

```powershell
winver
```

Expected finding in this case:

* Windows Server / Hyper-V Server 2016
* Version 1607
* Build 14393

### Why this matters

If the server is on build 14393, the VS Code terminal issue is expected with newer VS Code builds.

---

## Step 2: Try the VS Code terminal workaround

Open VS Code settings JSON and make sure this setting exists:

```json
"terminal.integrated.windowsUseConptyDll": true
```

### How to do it

1. Open VS Code
2. Press `Ctrl + Shift + P`
3. Search for:

   ```text
   Preferences: Open User Settings (JSON)
   ```
4. Add:

   ```json
   "terminal.integrated.windowsUseConptyDll": true
   ```
5. Save the file
6. Fully close all VS Code windows
7. Reopen VS Code
8. Test the integrated terminal

### Important note

This is only a workaround. On older Windows builds, it may still fail.

---

## Step 3: If the terminal still fails, downgrade VS Code

If the integrated terminal still does not open after adding the setting, do not keep troubleshooting random terminal settings.

The practical fix is to install an older VS Code version that is compatible with this OS.

### Recommended action

* Uninstall the current VS Code version
* Install **VS Code 1.108.2**
* Disable auto-update so it does not upgrade again automatically

### How to uninstall

1. Open **Add or Remove Programs**
2. Find **Microsoft Visual Studio Code**
3. Click **Uninstall**

### Optional cleanup

If a clean reset is needed, remove leftover user config directories after uninstall.

### Disable auto-update after reinstall

Add this to `settings.json`:

```json
"update.mode": "none"
```

### Why this matters

If auto-update stays enabled, the machine may upgrade back to a newer VS Code version and the terminal issue will return.

---

## Step 4: Use external PowerShell while VS Code terminal is unstable

If VS Code integrated terminal is not working yet, do not block progress.

Use **Administrator Windows PowerShell** outside of VS Code.

### Why this helps

* It avoids the broken VS Code integrated terminal path
* It allows installation and verification work to continue
* It is the safest path on older server environments

---

## Step 5: Fix the TLS error before installing `uv`

Open PowerShell and run:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

Then run the installer:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### What this does

* First line forces the current PowerShell session to use TLS 1.2
* Second line downloads and runs the `uv` installer script

### Important detail

This TLS setting is **session-only**.

That means:

* it works in the current PowerShell window
* it does **not** automatically carry over to a new PowerShell process launched later
* VS Code tasks that start a new shell may still fail unless TLS is set again in that new shell

---

## Step 6: Install `uv` machine-wide for all admins/users

If the server is shared, install `uv` in a machine-wide location instead of a single user profile.

### Recommended install location

```text
C:\Program Files\uv
```

### Machine-wide installation approach

In Administrator PowerShell:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
$env:UV_INSTALL_DIR = "C:\Program Files\uv"
$env:UV_NO_MODIFY_PATH = "1"
irm https://astral.sh/uv/install.ps1 | iex
```

Then add it to the system PATH if needed.

### Why this matters

A user-level install usually lands under:

```text
C:\Users\<username>\.local\bin
```

That is not ideal for a shared server because:

* it ties the tool to one user profile
* other admins may not get the same resolution path
* it creates duplicate installs later

---

## Step 7: Verify the `uv` installation

After installation, verify both version and path.

Run:

```powershell
uv --version
where.exe uv
```

### Expected result

Preferred output:

```text
uv 0.10.x
C:\Program Files\uv\uv.exe
```

### How to interpret results

#### Case 1: Only Program Files appears

Good. This is the cleanest result.

#### Case 2: Both Program Files and user profile paths appear

Example:

```text
C:\Program Files\uv\uv.exe
C:\Users\dsmchowdhu\.local\bin\uv.exe
```

This means there are two installs:

* a machine-wide install
* an older user-level install

If `Program Files` appears first, Windows is using the machine-wide one first.

### Recommended cleanup

If the machine-wide install works, remove the old user-level copy later to avoid confusion.

---

## Step 8: Install Python using `uv`

Once `uv` is working, install Python.

### Recommended command

```powershell
uv python install 3.12 --default
python --version
```

### What this does

* installs Python 3.12 through `uv`
* sets the default executable shims
* allows `python` to be called directly

### Validation

Run:

```powershell
python --version
uv python list
```

---

## Step 9: Create a project-level VS Code task as a fallback

Even after fixing most of the environment, using a task that directly calls the full `uv.exe` path is a stable approach.

### Why use a task

* avoids reinstall attempts
* avoids depending on PATH resolution during troubleshooting
* gives a repeatable project-level check

### Create `.vscode\tasks.json`

Inside the project folder, create:

```text
.vscode\tasks.json
```

Use this content:

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

### How to run it

1. Open the project in VS Code
2. Press `Ctrl + Shift + P`
3. Search for:

   ```text
   Tasks: Run Task
   ```
4. Select:

   ```text
   uv version
   ```

### Expected output

```text
uv 0.10.x
```

---

## Special note: Why TLS error may still happen in tasks

If a VS Code task tries to run this command again:

```powershell
irm https://astral.sh/uv/install.ps1 | iex
```

it may fail even though manual install succeeded earlier.

### Reason

VS Code tasks often launch a **new PowerShell process**.
That new shell does not inherit the prior session’s TLS 1.2 setting.

### Correct approach

Do **not** keep reinstalling `uv` in tasks.
Instead, call the installed binary directly:

```json
"command": "C:\\Program Files\\uv\\uv.exe"
```

---

## Validation checklist

Use this checklist at the end of the fix:

### OS / platform validation

* [ ] `winver` confirms the machine build
* [ ] engineer understands whether the OS is older than build 1809

### VS Code validation

* [ ] `terminal.integrated.windowsUseConptyDll` is present in `settings.json`
* [ ] VS Code was fully restarted after changing settings
* [ ] if terminal still failed, VS Code was downgraded
* [ ] `update.mode` was set to `none` after downgrade

### `uv` validation

* [ ] `uv --version` returns a valid version
* [ ] `where.exe uv` resolves to `C:\Program Files\uv\uv.exe`
* [ ] duplicate user install paths were identified if present

### Python validation

* [ ] `uv python install 3.12 --default` completed
* [ ] `python --version` works

### Project validation

* [ ] `.vscode\tasks.json` exists in the project
* [ ] the `uv version` task runs successfully

---

## Recommended final state

For this class of server, the preferred steady-state setup is:

* Windows Server 2016 build 14393 acknowledged as legacy platform
* VS Code **1.108.2** if newer versions break integrated terminal
* `update.mode` set to `none`
* `uv` installed machine-wide under:

  ```text
  C:\Program Files\uv
  ```
* old user-level `uv` install removed if no longer needed
* Python installed through `uv`
* project tasks call `C:\Program Files\uv\uv.exe` directly during stabilization

---

## Known pitfalls

### Pitfall 1: Treating VS Code setting like a PowerShell command

This is wrong:

```powershell
terminal.integrated.windowsUseConptyDll
```

That is not a shell command. It is a VS Code JSON setting.

### Pitfall 2: Editing the wrong `.vscode` folder

Do not edit files under the VS Code installation path like:

```text
C:\Program Files\Microsoft VS Code\resources\app\...
```

Use the `.vscode` folder inside the **project**.

### Pitfall 3: Thinking `uv` is broken when TLS is the real problem

If the installer fails with SSL/TLS channel error, the download failed before `uv` even had a chance to install.

### Pitfall 4: Reinstalling `uv` repeatedly in tasks

Once `uv` is installed, tasks should call the installed executable, not rerun the web installer every time.

### Pitfall 5: Keeping both user-level and machine-wide installs without noticing

This can confuse future engineers because `where.exe uv` may return more than one path.

---

## Troubleshooting quick reference

### Check Windows version

```powershell
winver
```

### Set TLS 1.2 for current session

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

### Install `uv`

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### Verify `uv`

```powershell
uv --version
where.exe uv
```

### Install Python

```powershell
uv python install 3.12 --default
python --version
```

### Example fallback task command

```json
"command": "C:\\Program Files\\uv\\uv.exe"
```

---

## Escalation guidance

Escalate if any of the following happens:

* VS Code 1.108.2 still cannot launch terminal on this machine
* machine-wide PATH cannot be updated due to policy restrictions
* TLS 1.2 setting does not resolve HTTPS download failures
* `uv` installs but `python` installation fails repeatedly
* another security tool or corporate policy blocks script download or execution

When escalating, provide:

* screenshot of `winver`
* output of `uv --version`
* output of `where.exe uv`
* current VS Code version
* whether integrated terminal works
* whether external PowerShell works

---

## Long-term recommendation

This entire issue chain is a sign that the server is running on an aging Windows platform.

The long-term engineering fix is to move to a newer Windows version that supports the modern terminal stack required by recent VS Code releases. Until then, use the downgrade + machine-wide install + direct task execution approach documented above.
