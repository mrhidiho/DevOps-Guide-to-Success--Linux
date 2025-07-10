
# Chapter1  Strict Mode & Core Terminology

_Fail fast, fail loud, fail **before** you flash 5,000 modems with a bad build._

This chapter explains every shell option and special parameter you need to
run **bulletproof Bash** in a communicationsindustry DevOps role.

---

## 1.1  Shebangs and POSIX vs Bash

| Shebang | When to use | Why it matters in fieldgear |
|---------|-------------|------------------------------|
| `#!/usr/bin/env bash` | Scripts that **require** Bash features (arrays, `[[ ]]`, `${var^^}`) | Portable: `env` finds Bash even on BusyBox or custom firmware. |
| `#!/bin/sh` | Scripts that _must_ run on the tiniest BusyBox/Alpine rescue shell | Limits you to POSIX syntax; no arrays. |

> **Tip:** When in doubt, make two scripts: a tiny `/bin/sh` bootstrap that
> _fetches_ your full Bash toolkit and runs it once Bash is confirmed.

---

## 1.2  Enabling StrictMode

### Full header (copy/paste)

```bash
#!/usr/bin/env bash
set -euo pipefail          # strict mode flags
IFS=$'\n\t'               # safe word splitting
shopt -s inherit_errexit   # propagate -e into subshells (bash4.4+)

LOG=/var/log/$(basename "$0").log
exec 3>&1 1>>"$LOG" 2>&1   # stdout+stderr  log, FD3 = console
```

### What each flag does

| Flag | Behaviour | Real failure it prevents |
|------|-----------|--------------------------|
| `-e` | Exit immediately if **any** command returns nonzero. | `scp` fails midloop  script stops before `fwupdate` with corrupt file. |
| `-u` | Treat **unset vars** as an error. | Mistyped `$DEST_PATH` bricks only one node instead of many. |
| `pipefail` | A pipeline returns the first nonzero in the chain. | `grep 'OK' log | tail -1` no longer hides a failed `grep`. |
| `IFS=$'\n\t'` | Wordsplitting only on newline / tab. | Hostnames with spaces (`"lab switch"`) no longer break loops. |
| `inherit_errexit` | `-e` applies in subshells & functions run by `bash -c`. | Backgrounded `ssh` failures propagate up to main script. |

---

## 1.3  Special Parameters CheatSheet

| Symbol | Meaning | Example |
|--------|---------|---------|
| `$?` | Exit status of last command | `cmd; [[ $? -ne 0 ]] && log "fail"` |
| `$#` | Number of script args | `(( $# < 1 )) && { echo "usage"; exit 2; }` |
| `$@` | All args, **quoted individually** | `for file in "$@"; do ` |
| `$*` | All args as **single word** | `printf '%s\n' $*` |
| `$-` | Current shell flags | Useful in debug wrappers. |
| `$BASH_SOURCE`, `$LINENO` | Current file & line (inside functions) | For precise error logging. |

---

## 1.4  Reserved Words vs Builtins vs External

| Category | Example | Why you care |
|----------|---------|--------------|
| **Reserved** | `if`, `case`, `for`, `[[` | Implemented by parser, no fork. |
| **Builtin** | `echo`, `printf`, `read`, `test` | Faster/no fork matters in 50knode loops. |
| **External** | `/bin/grep`, `/usr/bin/sed` | Spawns process; use sparingly inside tight loops. |

Use `type -a cmd` to see which class a token belongs to.

---

## 1.5  Useful `shopt` Options

| Option | Default | What it does | Commsindustry usecase |
|--------|---------|--------------|-------------------------|
| `globstar` | off | `**` matches recursive dirs | Collect `*.cfg` across firmware trees quickly. |
| `nocaseglob` | off | Globs ignore case | Vendor dumps files as `CONFIG.TXT` vs `config.txt`. |
| `histverify` | off | Show expanded history cmd before execution | Prevent fatfinger accidents on prod jumpbox. |

Enable with `shopt -s globstar`.

---

## 1.6  ExitCode Strategy

| Code | Meaning | Example in scripts |
|------|---------|--------------------|
| 0 | success | `return 0` after all nodes flashed |
| 1 | generic failure | Any unhandled command error |
| 2 | incorrect usage | Missing required arg (`--fw`) |
| 3 | partial success | e.g., 2 of 500 nodes failed flash |
| 126 | cmd not executable | BusyBox missing binary |
| 127 | cmd not found | Typos (`scp1`) |
| 130 | Interrupted (CtrlC) | Used by `timeout` on SIGINT |

A CI pipeline can act differently on 3 (retry) vs 1 (abort).

---

## 1.7  RealWorld Failure Scenario

*Problem:*  
A junior tech uploads `fw.bin` to `/tmp` but the copy silently fails on three
edge routers (disk full). Without strictmode the script continues and calls
`fwupdate apply`, leaving devices in a **partial** state.

*With strictmode enabled* the failed `scp` exits nonzero  script aborts
before the apply step. Only the failing devices are touched, and you can
recover with freespace cleanup.

---

### 1.8  Recommended Boilerplate (copyready)

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
shopt -s inherit_errexit

trap 'echo "[ERR] ${BASH_SOURCE[0]}:${LINENO}: $BASH_COMMAND" >&2' ERR

log(){ printf "[%(%F %T)T] %s\n" -1 "$*" >&2; }
cleanup(){ log "cleanup executed"; }
trap cleanup EXIT
```

*Place this at the top of every fleetautomation script.*

---

## 1.9  Exercises

1. Turn a silent failure (`grep 'OK' file | tail -1`) into a pipefail demo.  
2. Write a script that **counts** how many times `$-` contains `e` during nested
   function callsobserve with and without `inherit_errexit`.  
3. Create a loop that reads `devices.csv` and deliberately references
   `${UNKNOWN}`verify `-u` stops the script before any SSH.

---

> **Next chapter:** Variables, Arrays & Parameter Expansion.

