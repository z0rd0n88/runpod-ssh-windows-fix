# Fixing RunPod SSH from Windows in Non-Terminal Contexts

**A deep dive into why `ssh` to RunPod breaks from Git Bash, CI/CD pipelines, and AI coding agents on Windows &mdash; and the three-layer fix.**

> **TL;DR** &mdash; RunPod's SSH gateway is a custom Go server that requires PTY allocation and ignores standard SSH exec-mode commands. On Windows, Git Bash's MSYS2-compiled SSH client cannot allocate a PTY when there is no controlling terminal &mdash; which is the case in CI/CD runners, cron jobs, background scripts, automation tools, and AI coding agents like Claude Code, Cursor, Windsurf, and GitHub Copilot. The fix: call `C:\Windows\System32\OpenSSH\ssh.exe` directly with `-tt` and pipe commands through stdin.

---

## This Is Not a Claude Code Bug

This issue was discovered while using [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (Anthropic's AI coding agent), but **the root cause has nothing to do with Claude Code**. It affects any process on Windows that:

1. Invokes SSH without a controlling terminal (no TTY)
2. Uses Git Bash's bundled SSH client (which most Windows dev tools do)
3. Connects to RunPod's non-standard SSH gateway

**You will hit this same error in:**

- **CI/CD pipelines** &mdash; GitHub Actions, GitLab CI, Jenkins, Azure DevOps running on Windows runners
- **AI coding agents** &mdash; Claude Code, Cursor, Windsurf, Cline, Aider, GitHub Copilot, or any tool that shells out to `ssh` on Windows
- **Task automation** &mdash; Windows Task Scheduler, cron-like tools, background PowerShell scripts
- **Process managers** &mdash; PM2, supervisor, or any daemon that spawns SSH as a subprocess
- **IDE terminals (sometimes)** &mdash; Some VS Code terminal configurations don't provide a proper TTY to subprocesses
- **Docker for Windows** &mdash; Containers running Git Bash or MSYS2 toolchains

If you've ever seen `Error: Your SSH client doesn't support PTY` when connecting to RunPod from Windows, this writeup is for you &mdash; regardless of what tool triggered it.

---

## Table of Contents

- [This Is Not a Claude Code Bug](#this-is-not-a-claude-code-bug)
- [The Problem](#the-problem)
- [Environment](#environment)
- [Root Cause Analysis](#root-cause-analysis)
  - [Layer 1: Git Bash's SSH vs Windows-native SSH](#layer-1-git-bashs-ssh-vs-windows-native-ssh)
  - [Layer 2: RunPod's Custom Go SSH Gateway](#layer-2-runpods-custom-go-ssh-gateway)
  - [Layer 3: No Controlling Terminal (The Universal Trigger)](#layer-3-no-controlling-terminal-the-universal-trigger)
- [The Debugging Journey](#the-debugging-journey)
  - [Attempt 1: Direct SSH from Git Bash](#attempt-1-direct-ssh-from-git-bash)
  - [Attempt 2: PowerShell Wrapper with -T](#attempt-2-powershell-wrapper-with--t)
  - [Attempt 3: Debug Trace Reveals the Truth](#attempt-3-debug-trace-reveals-the-truth)
  - [Attempt 4: Force PTY with -t](#attempt-4-force-pty-with--t)
  - [Attempt 5: stdin Pipe with -tt](#attempt-5-stdin-pipe-with--tt)
  - [Attempt 6: Drop the PowerShell Wrapper](#attempt-6-drop-the-powershell-wrapper)
- [The Solution](#the-solution)
  - [Command Pattern](#command-pattern)
  - [Why Each Piece Matters](#why-each-piece-matters)
- [Who Is Affected](#who-is-affected)
- [Claude Code Skill Implementation](#claude-code-skill-implementation)
  - [What Is a Claude Code Skill?](#what-is-a-claude-code-skill)
  - [File Location](#file-location)
  - [Full Skill File](#full-skill-file)
- [Adapting This for Your Setup](#adapting-this-for-your-setup)
- [Known Limitations](#known-limitations)
- [Key Takeaways](#key-takeaways)

---

## The Problem

Running SSH commands on a RunPod pod from any non-terminal context on Windows fails immediately:

```
$ ssh -i /c/Users/youruser/.ssh/id_ed25519 \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    -o LogLevel=ERROR \
    -T "your-pod-id-here@ssh.runpod.io" \
    supervisorctl status

Error: Your SSH client doesn't support PTY
```

The same command works perfectly when typed directly into a PowerShell or Git Bash terminal window. It works from WSL. It works anywhere there's an interactive terminal. It **only** fails when SSH is invoked from a process that doesn't have a controlling TTY &mdash; which includes CI/CD runners, background scripts, automation tools, and AI coding agents.

---

## Environment

| Component | Version / Details |
|---|---|
| OS | Windows 11 |
| Claude Code Bash tool | Runs inside Git Bash (MSYS2) |
| Git Bash SSH | `/usr/bin/ssh` &mdash; `OpenSSH_10.2p1, OpenSSL 3.5.5` |
| Windows-native SSH | `C:\Windows\System32\OpenSSH\ssh.exe` &mdash; `OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2` |
| RunPod SSH gateway | Custom Go-based SSH server (`remote software version Go`) |
| SSH key | Ed25519 at `C:\Users\youruser\.ssh\id_ed25519` |

---

## Root Cause Analysis

This is not one bug. It's **three independent issues** that are invisible in isolation and only manifest when all three conditions are present simultaneously. You could use RunPod for months from an interactive terminal and never see this. The moment you automate it, everything breaks.

```
                              ┌─────────────────────┐
                              │   "Works on my       │
                              │    terminal"          │
                              └──────────┬────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
   ┌──────────▼──────────┐  ┌───────────▼──────────┐  ┌───────────▼──────────┐
   │  Layer 1             │  │  Layer 2              │  │  Layer 3              │
   │  Two SSH clients     │  │  RunPod's Go gateway  │  │  No controlling TTY   │
   │  on Windows PATH     │  │  requires PTY +       │  │  in automated/CI      │
   │  (Git vs native)     │  │  ignores exec-mode    │  │  contexts              │
   └──────────────────────┘  └───────────────────────┘  └───────────────────────┘
              │                          │                          │
              └──────────────────────────┼──────────────────────────┘
                                         │
                              ┌──────────▼────────────┐
                              │   "Your SSH client     │
                              │    doesn't support     │
                              │    PTY"                │
                              └────────────────────────┘
```

All three must be addressed. Fixing any single layer still leaves you broken.

### Layer 1: Git Bash's SSH vs Windows-native SSH

Windows has **two completely independent SSH clients** installed by default on most developer machines:

```
# Git Bash's bundled SSH (MSYS2/MinGW)
$ which ssh
/usr/bin/ssh

$ ssh -V
OpenSSH_10.2p1, OpenSSL 3.5.5

# Windows-native SSH
$ powershell.exe -Command "& 'C:\Windows\System32\OpenSSH\ssh.exe' -V"
OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2
```

Claude Code's Bash tool runs inside Git Bash. Every `ssh` call uses Git's MSYS2-compiled binary, not Windows' native one.

**Critical trap**: Even PowerShell's `Get-Command ssh` resolves to Git's copy, because Git adds itself to the system PATH before `C:\Windows\System32\OpenSSH`:

```powershell
PS> Get-Command ssh | Select-Object Source

Source
------
C:\Program Files\Git\usr\bin\ssh.exe    # <-- Git's copy, not Windows'
```

This means `powershell.exe -Command "ssh ..."` **still uses the broken Git SSH**. You must reference the Windows binary by its full absolute path: `C:\Windows\System32\OpenSSH\ssh.exe`.

### Layer 2: RunPod's Custom Go SSH Gateway

RunPod does not run a standard OpenSSH server. Their SSH gateway is a custom Go application:

```
debug1: Remote protocol version 2.0, remote software version Go
debug1: compat_banner: no match: Go
```

This custom gateway has two non-standard behaviors:

1. **It requires PTY allocation.** Standard SSH servers happily execute commands without a PTY (via the "exec" channel type). RunPod's gateway rejects connections that don't request a PTY, responding with `Error: Your SSH client doesn't support PTY`.

2. **It ignores SSH exec-mode commands.** In standard SSH, you can pass a command as an argument (`ssh host "uptime"`) and the server executes it via an exec channel request. RunPod's gateway ignores this entirely. It always opens an interactive shell session, regardless of whether a command argument is provided.

### Layer 3: No Controlling Terminal (The Universal Trigger)

This is the layer that makes the bug appear or disappear depending on how you invoke SSH. It's also why "works on my terminal" is technically true and completely unhelpful.

When you type `ssh` in a PowerShell window or Git Bash terminal, the process has a **controlling TTY** &mdash; a real terminal that the SSH client can use to set up PTY forwarding. When SSH is invoked by an automated process (CI runner, cron job, subprocess spawn, AI agent), there is **no controlling TTY**.

This matters because:

- SSH's `-t` flag requests PTY allocation from the remote server, but the client needs a local terminal to map it to
- Without a local TTY, the SSH client cannot set up the PTY properly
- Git Bash's MSYS2-compiled SSH is especially strict about this &mdash; it flat-out refuses to allocate a PTY when it detects no local terminal
- Most SSH servers don't care (they support exec-mode without PTY), so this never surfaces. RunPod's gateway **does** care, so it surfaces immediately.

**The universal trigger is: no TTY + RunPod.** The specific tool that invokes SSH (Claude Code, GitHub Actions, Jenkins, a Python subprocess, a Node.js `child_process.exec`) is irrelevant. They all produce the same error for the same reason.

The `-tt` flag (double-t) **forces** PTY allocation even when there is no local terminal. This is the key that makes the Windows-native SSH client work from non-terminal contexts.

---

## The Debugging Journey

Here's every approach that was tried, what happened, and why it failed or succeeded.

### Attempt 1: Direct SSH from Git Bash

**Command:**
```bash
ssh -T -i /c/Users/youruser/.ssh/id_ed25519 \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    -o LogLevel=ERROR \
    "your-pod-id-here@ssh.runpod.io" \
    "supervisorctl status"
```

**Result:**
```
Error: Your SSH client doesn't support PTY
```

**Why it failed:** Git Bash's SSH sent the command via exec channel with no PTY. RunPod's gateway rejected it because it requires PTY allocation.

---

### Attempt 2: PowerShell Wrapper with -T

**Hypothesis:** Git Bash's SSH is broken; use the Windows-native SSH through PowerShell.

**Command:**
```bash
powershell.exe -Command "& 'C:\Windows\System32\OpenSSH\ssh.exe' \
    -T -i 'C:\Users\youruser\.ssh\id_ed25519' \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=NUL \
    -o LogLevel=ERROR \
    -p 22 'your-pod-id-here@ssh.runpod.io' \
    'supervisorctl status'"
```

**Result:**
```
Error: Your SSH client doesn't support PTY
```

**Why it failed:** We confirmed the correct binary was running (`OpenSSH_for_Windows_9.5p2`), but `-T` explicitly disables PTY allocation. RunPod's gateway still rejected it. The error comes from the **server**, not the client.

---

### Attempt 3: Debug Trace Reveals the Truth

**Command:** Same as Attempt 2 but with `-o LogLevel=DEBUG3`.

**Key findings from the 80+ line debug trace:**

```
debug1: Remote protocol version 2.0, remote software version Go
debug1: Sending command: echo hello
debug2: exec request accepted on channel 0
debug1: client_input_channel_req: channel 0 rtype exit-status reply 0
Error: Your SSH client doesn't support PTY
debug1: Exit status 0
```

This was the breakthrough. The SSH connection **succeeded** &mdash; authentication passed, the exec request was accepted, exit status was 0. But the gateway returned the PTY error as the command's output instead of executing the command. The error is not an SSH protocol failure; it's application-level behavior from RunPod's custom gateway.

---

### Attempt 4: Force PTY with -t

**Hypothesis:** RunPod needs a PTY. Give it one.

**Command:**
```bash
powershell.exe -Command "& 'C:\Windows\System32\OpenSSH\ssh.exe' \
    -t -i 'C:\Users\youruser\.ssh\id_ed25519' \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=NUL \
    -o LogLevel=ERROR \
    -p 22 'your-pod-id-here@ssh.runpod.io' \
    'supervisorctl status'"
```

**Result:**
```
-- RUNPOD.IO --
Enjoy your Pod #your-pod-id ^_^

root@abc123def456:/app#
```

**Partial success!** The PTY was allocated and the connection succeeded. But the command argument (`supervisorctl status`) was completely ignored. RunPod's gateway opened an interactive shell instead of executing the command. The session hung waiting for interactive input.

This confirmed RunPod's gateway Layer 2 behavior: exec-mode commands are ignored; only shell sessions are supported.

---

### Attempt 5: stdin Pipe with -tt

**Hypothesis:** If exec-mode doesn't work, pipe commands through stdin to the interactive shell.

**Command:**
```bash
printf 'supervisorctl status\nexit\n' | \
    powershell.exe -Command "& 'C:\Windows\System32\OpenSSH\ssh.exe' \
    -tt -i 'C:\Users\youruser\.ssh\id_ed25519' \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=NUL \
    -o LogLevel=ERROR \
    -p 22 'your-pod-id-here@ssh.runpod.io'" 2>&1
```

**Result:**
```
-- RUNPOD.IO --
Enjoy your Pod #your-pod-id ^_^

[?2004hroot@abc123def456:/app# supervisorctl status
[?2004lunix:///var/run/supervisor.sock no such file
[?2004hroot@abc123def456:/app# exit
[?2004lexit
Connection to 10.x.x.x closed.
```

**Success!** The command executed on the remote pod and returned real output (`unix:///var/run/supervisor.sock no such file` &mdash; supervisord wasn't running at the time, but that's a valid response). The session exited cleanly.

The output is noisy (ANSI escape codes, shell prompts, banner), but the actual command result is clearly present.

**Why `-tt` instead of `-t`:** A single `-t` requests PTY allocation but may still fail if the client detects no local terminal. Double `-tt` **forces** PTY allocation unconditionally, even when stdin is not a terminal. This is critical when running from Claude Code's non-terminal Bash context.

---

### Attempt 6: Drop the PowerShell Wrapper

**Hypothesis:** If we're calling the Windows SSH binary by absolute path, do we even need the PowerShell wrapper? Can Git Bash invoke it directly?

**Command:**
```bash
printf 'hostname 2>&1\nexit\n' | \
    MSYS_NO_PATHCONV=1 \
    /c/Windows/System32/OpenSSH/ssh.exe \
    -tt -i "C:\Users\youruser\.ssh\id_ed25519" \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=NUL \
    -o LogLevel=ERROR \
    -p 22 "your-pod-id-here@ssh.runpod.io" 2>&1
```

**Result:**
```
-- RUNPOD.IO --
Enjoy your Pod #your-pod-id ^_^

root@abc123def456:/app# hostname
abc123def456
root@abc123def456:/app# exit
Connection to 10.x.x.x closed.
```

**Success, and simpler!** Git Bash can invoke the Windows SSH binary directly using its MSYS2-translated path (`/c/Windows/System32/OpenSSH/ssh.exe`). No PowerShell wrapper needed.

The `MSYS_NO_PATHCONV=1` environment variable is critical here &mdash; it prevents Git Bash from mangling the Windows-style paths in the SSH arguments (like the key path and `UserKnownHostsFile=NUL`).

---

## The Solution

### Command Pattern

```bash
printf 'REMOTE_COMMAND 2>&1\nexit\n' | \
    MSYS_NO_PATHCONV=1 \
    /c/Windows/System32/OpenSSH/ssh.exe \
    -tt \
    -i "C:\Users\youruser\.ssh\id_ed25519" \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=NUL \
    -o LogLevel=ERROR \
    -p 22 \
    "USER@ssh.runpod.io" \
    2>&1
```

Replace `REMOTE_COMMAND` with whatever you want to run on the pod, and `USER` with your RunPod SSH user identifier.

### Why Each Piece Matters

| Component | Purpose | What happens without it |
|---|---|---|
| `printf '...\nexit\n'` | Pipes commands to the remote shell's stdin, then sends `exit` to close the session | SSH session hangs indefinitely waiting for interactive input |
| `MSYS_NO_PATHCONV=1` | Disables Git Bash's automatic MSYS2 path translation | Git Bash converts `C:\Users\...` to `/c/Users/...` and `NUL` to `/nul`, breaking the Windows SSH arguments |
| `/c/Windows/System32/OpenSSH/ssh.exe` | Uses the Windows-native OpenSSH binary, not Git's MSYS2 copy | Git's SSH (`/usr/bin/ssh`) cannot allocate PTY from a non-terminal context |
| `-tt` | Forces PTY allocation even without a local terminal (double-t) | RunPod's gateway rejects the connection: "Your SSH client doesn't support PTY" |
| `-i "C:\Users\...\id_ed25519"` | SSH key path in **Windows format** | The Windows-native SSH binary doesn't understand MSYS2 paths like `/c/Users/...` |
| `-o UserKnownHostsFile=NUL` | Suppresses host key verification using Windows' null device | `/dev/null` doesn't exist from the perspective of a Windows-native binary; SSH errors or prompts for host key verification |
| `-o StrictHostKeyChecking=no` | Auto-accepts RunPod's host key | SSH prompts for confirmation, which blocks in a non-interactive context |
| `-o LogLevel=ERROR` | Suppresses SSH info/warning messages | Output is polluted with SSH negotiation messages mixed into the command results |
| `2>&1` (outer) | Captures stderr alongside stdout | Error messages from SSH or remote commands are silently lost |

---

## Who Is Affected

The underlying issue &mdash; RunPod's gateway requiring PTY from a client that has no terminal &mdash; is platform and tool agnostic. Here's a concrete breakdown:

### Definitely affected (Windows)

| Context | Why | Uses Git SSH? |
|---|---|---|
| **Claude Code** | Bash tool runs in Git Bash (MSYS2), no TTY | Yes |
| **Cursor / Windsurf / Cline** | Terminal subprocess, no TTY | Depends on config |
| **GitHub Actions (Windows runner)** | `run:` steps have no TTY | Yes, if Git is on PATH |
| **GitLab CI (Windows runner)** | Shell executor, no TTY | Yes, if Git is on PATH |
| **Jenkins (Windows agent)** | Build steps have no TTY | Yes, if Git is on PATH |
| **Windows Task Scheduler** | Background process, no TTY | Yes, if Git is on PATH |
| **Python `subprocess.run("ssh ...")`** | Child process, no TTY | Inherits PATH |
| **Node.js `child_process.exec`** | Child process, no TTY | Inherits PATH |
| **PowerShell `Start-Process`** | Depends on `-NoNewWindow` | No (uses Windows SSH) |

### Not affected

| Context | Why |
|---|---|
| **Interactive PowerShell terminal** | Has a controlling TTY |
| **Interactive Git Bash terminal** | Has a controlling TTY |
| **WSL** | Linux SSH binary, has TTY, no MSYS2 issues |
| **macOS / Linux terminal** | Native SSH, has TTY |
| **macOS / Linux CI/CD** | Native SSH + `-tt` is usually enough (no dual-SSH-client issue) |

### The "it works for me" trap

This is why the bug is so insidious. Every developer testing SSH to RunPod does so from an interactive terminal, where it works. The failure only appears in automation &mdash; the exact context where you're least likely to be watching the output in real time. The error message (`Your SSH client doesn't support PTY`) sounds like a client misconfiguration, leading developers to chase SSH client upgrades, key permissions, and config file changes &mdash; none of which are the actual problem.

---

## Claude Code Skill Implementation

### What Is a Claude Code Skill?

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) is a markdown file that teaches Claude how to perform a specific task. When the user's request matches the skill's description, Claude loads the skill instructions and follows them. Skills are placed in:

- **Project scope:** `<project>/.claude/skills/<name>/SKILL.md` &mdash; available only in that project
- **Global scope:** `~/.claude/skills/<name>/SKILL.md` &mdash; available in every Claude Code session

Skills are the right abstraction for this problem because the SSH workaround involves environment-specific knowledge (which binary to call, path formats, flags) that Claude wouldn't know on its own.

### File Location

```
~/.claude/skills/runpod-ssh/SKILL.md
```

On Windows, `~/.claude/` translates to `C:\Users\<username>\.claude\`.

### Full Skill File

<details>
<summary>Click to expand <code>SKILL.md</code></summary>

```markdown
---
name: runpod-ssh
description: Run SSH commands on the RunPod pod from Windows. Use when the user asks about pod status, logs, SSH, RunPod, supervisorctl, ollama on the pod, restarting services, or any remote command on the deployment.
---

When the user asks to run a command on the RunPod pod, check pod status, tail logs,
restart services, or perform any SSH-based remote operation:

## 1. Read connection details from `.env.runpod`

Read the `.env.runpod` file in the project root to extract:
- `RUNPOD_SSH_USER` (e.g., `your-pod-id-here`)
- `RUNPOD_SSH_KEY` (e.g., `/c/Users/youruser/.ssh/id_ed25519`)
- `RUNPOD_SSH_HOST` (default: `ssh.runpod.io`)
- `RUNPOD_SSH_PORT` (default: `22`)

If `.env.runpod` does not exist, tell the user to copy `.env.runpod.example`
and fill in values.

## 2. Convert the SSH key path from Git Bash to Windows format

The key path in `.env.runpod` uses Git Bash (MSYS2) format. Convert it to a
Windows path:
- `/c/Users/...` becomes `C:\Users\...`
- `/d/something/...` becomes `D:\something\...`
- General rule: replace leading `/<drive-letter>/` with `<DRIVE-LETTER>:\`
  and all `/` with `\`
- If the path is already Windows format (`C:\...`), leave it as-is.

## 3. Execute using Windows-native OpenSSH via stdin pipe

**WHY THIS APPROACH**: RunPod's SSH gateway is a custom Go-based server that:
- REQUIRES PTY allocation (rejects connections without it)
- Only supports shell sessions (ignores SSH exec-mode command arguments)
- Git Bash's bundled SSH client cannot allocate PTY from Claude Code's
  non-terminal context

**CRITICAL RULES**:
- Do NOT use Git Bash's `ssh` (`/usr/bin/ssh`) -- it cannot allocate PTY
  without a terminal
- Do NOT pass commands as SSH arguments -- RunPod ignores exec-mode, opens
  interactive shell instead
- MUST use `-tt` (double-t) to force PTY allocation without a local terminal
- MUST pipe commands via `printf` through stdin
- MUST use `MSYS_NO_PATHCONV=1` to prevent Git Bash from mangling Windows paths
- MUST use `UserKnownHostsFile=NUL` (not `/dev/null`) -- this is Windows-native SSH

Use this exact pattern in the Bash tool:

    printf 'REMOTE_COMMAND 2>&1\nexit\n' | MSYS_NO_PATHCONV=1 \
        /c/Windows/System32/OpenSSH/ssh.exe -tt \
        -i "WINDOWS_KEY_PATH" \
        -o StrictHostKeyChecking=no \
        -o UserKnownHostsFile=NUL \
        -o LogLevel=ERROR \
        -p PORT "USER@HOST" 2>&1

**Reading the output**: The raw output will include RunPod's banner
(`-- RUNPOD.IO --`, `Enjoy your Pod`), shell prompts (`root@...:/app#`),
ANSI escape codes, and command echoes mixed with the actual command output.
Ignore the noise and extract the meaningful result.

## 4. Common operations

| User says                          | Remote command                                    |
|------------------------------------|---------------------------------------------------|
| "check pod status" / "status"      | `supervisorctl status`                            |
| "restart services" / "restart pod" | `supervisorctl restart all`                       |
| "restart bot"                      | `supervisorctl restart bot`                       |
| "restart model"                    | `supervisorctl restart model`                     |
| "pod logs" / "tail logs"           | `tail -n 50 /var/log/supervisor/supervisord.log`  |
| "bot logs"                         | `supervisorctl tail -200 bot stderr`              |
| "model logs"                       | `supervisorctl tail -200 model stderr`            |
| "ollama logs"                      | `supervisorctl tail -200 ollama stderr`           |
| "list models" / "ollama list"      | `ollama list`                                     |
| "health check"                     | `curl -sf http://localhost:8000/health`            |
| "disk space"                       | `df -h`                                           |
| "memory usage"                     | `free -h`                                         |
| custom command                     | Pass the user's command directly                  |

For compound requests (e.g., "restart bot and check status"), chain commands
with `&&` or `;` inside a single line in the printf.

## 5. Error handling

- If SSH hangs or times out, suggest the user verify the pod is running in
  the RunPod console
- If "Connection refused", the pod may be stopped or SSH service not running
- If "Permission denied", the SSH key may not match the pod's authorized_keys
- Always show the raw output to the user -- do not silently swallow errors
```

</details>

---

## Adapting This for Your Setup

The Claude Code skill above is one implementation of the fix. The underlying technique applies to **anyone on Windows who needs to SSH into RunPod from a non-terminal context** &mdash; regardless of what tool, language, or automation framework is invoking SSH.

### For Claude Code / AI coding agents

Use the skill file as shown above. Place it in `~/.claude/skills/runpod-ssh/SKILL.md` for global access or `<project>/.claude/skills/runpod-ssh/SKILL.md` for project scope.

### For CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins)

Add the Windows-native SSH binary to your pipeline explicitly. Don't rely on `ssh` from PATH:

```yaml
# GitHub Actions example (Windows runner)
- name: Check RunPod pod status
  shell: bash
  run: |
    printf 'supervisorctl status 2>&1\nexit\n' | \
      MSYS_NO_PATHCONV=1 \
      /c/Windows/System32/OpenSSH/ssh.exe \
      -tt -i "${{ secrets.SSH_KEY_PATH }}" \
      -o StrictHostKeyChecking=no \
      -o UserKnownHostsFile=NUL \
      -o LogLevel=ERROR \
      -p 22 "${{ secrets.RUNPOD_SSH_USER }}@ssh.runpod.io" 2>&1
```

### For Python scripts

```python
import subprocess

cmd = (
    'printf "supervisorctl status 2>&1\\nexit\\n" | '
    "MSYS_NO_PATHCONV=1 "
    "/c/Windows/System32/OpenSSH/ssh.exe "
    '-tt -i "C:\\Users\\you\\.ssh\\id_ed25519" '
    "-o StrictHostKeyChecking=no "
    "-o UserKnownHostsFile=NUL "
    "-o LogLevel=ERROR "
    '-p 22 "your-user@ssh.runpod.io"'
)
result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
print(result.stdout)
```

### For Node.js

```javascript
const { execSync } = require("child_process");

const remoteCmd = "supervisorctl status";
const output = execSync(
  `printf '${remoteCmd} 2>&1\\nexit\\n' | ` +
    `MSYS_NO_PATHCONV=1 /c/Windows/System32/OpenSSH/ssh.exe ` +
    `-tt -i "C:\\Users\\you\\.ssh\\id_ed25519" ` +
    `-o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL ` +
    `-o LogLevel=ERROR -p 22 "your-user@ssh.runpod.io"`,
  { encoding: "utf8", shell: "C:\\Program Files\\Git\\bin\\bash.exe" }
);
console.log(output);
```

### For standard SSH servers (not RunPod)

If your SSH server supports standard exec mode (most do &mdash; RunPod is the exception), you can simplify to just using the Windows SSH binary with `-tt` and passing commands as arguments:

```bash
MSYS_NO_PATHCONV=1 /c/Windows/System32/OpenSSH/ssh.exe -tt \
    -i "C:\Users\you\.ssh\id_ed25519" \
    user@your-server "your-command"
```

The stdin pipe + `exit` is only needed because RunPod's gateway ignores exec-mode.

### For Linux/macOS

You likely don't need any of this. The standard `ssh` binary with `-tt` should work fine. There's no dual-SSH-client problem, no MSYS2 path mangling, and no `NUL` vs `/dev/null` mismatch. If you're hitting the PTY error on Linux/macOS, just add `-tt`:

```bash
ssh -tt -i ~/.ssh/id_ed25519 user@ssh.runpod.io "your-command"

# If RunPod still ignores the command (exec-mode issue), use the pipe:
printf 'your-command 2>&1\nexit\n' | \
    ssh -tt -i ~/.ssh/id_ed25519 user@ssh.runpod.io
```

---

## Known Limitations

- **Noisy output.** The stdin-pipe approach produces output mixed with RunPod's banner, shell prompts, ANSI escape codes, and command echoes. Claude (the AI) is good at parsing through this, but it's not machine-friendly for scripted automation.

- **No streaming.** Long-running commands (like `tail -f`) will cause the Bash tool to hang until timeout. Use bounded alternatives (`tail -n 50`) instead.

- **Single-quotes in remote commands require escaping.** If your remote command contains single quotes, they need to be escaped within the `printf` string. For example:
  ```bash
  printf 'bash -c "echo '\''hello world'\''"\nexit\n' | ...
  ```

- **Session overhead.** Every command opens a new SSH connection, authenticates, receives the banner, and exits. There's ~1-2 seconds of overhead per invocation. This is acceptable for operational commands but would be slow for rapid-fire scripting.

- **Windows-native SSH must be installed.** The `C:\Windows\System32\OpenSSH\ssh.exe` binary ships with Windows 10 1809+ and Windows 11 by default, but may be missing on older or stripped-down installations. You can install it via Settings > Apps > Optional Features > OpenSSH Client.

---

## Key Takeaways

1. **This is not a tool-specific bug.** It's a platform-level interaction between Windows' dual SSH clients, RunPod's non-standard SSH gateway, and the absence of a controlling terminal. Any automation tool that invokes SSH on Windows will hit it.

2. **Windows has two SSH clients that behave differently.** Git Bash's MSYS2-compiled `ssh` and Windows' native `ssh.exe` at `C:\Windows\System32\OpenSSH\` are separate binaries with different PTY capabilities. Even `Get-Command ssh` in PowerShell can resolve to the wrong one because Git prepends itself to PATH.

3. **RunPod's SSH gateway is not standard OpenSSH.** It's a custom Go server that requires PTY and only supports shell sessions, not exec-mode commands. This breaks the standard `ssh host "command"` pattern that every SSH tutorial teaches.

4. **`-tt` is not the same as `-t`.** A single `-t` requests PTY but may be denied if there's no local terminal. Double `-tt` forces it unconditionally. This distinction matters in every non-interactive context: CI/CD, cron jobs, subprocess calls, and AI coding agents.

5. **`MSYS_NO_PATHCONV=1` is essential** when calling Windows-native binaries from Git Bash with path arguments. Without it, Git Bash's automatic MSYS2 path translation silently corrupts Windows-format paths, turning `NUL` into `/nul` and `C:\Users` into `/c/Users`.

6. **"Works on my terminal" is the root of the misdiagnosis.** Every developer who tests SSH to RunPod does so from an interactive terminal, where it works. The failure only appears in automation. The error message sounds like a client config issue, sending developers on wild goose chases through SSH versions, key permissions, and config files.

7. **The fix is three things, not one.** Use the Windows-native SSH binary (not Git's), force PTY with `-tt`, and pipe commands through stdin (because RunPod ignores exec-mode). Miss any one of these and it still breaks.

---

*Built and debugged live in a Claude Code session. Every attempt, error, and discovery documented as it happened.*
