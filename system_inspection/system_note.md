# System Note — Operational Guidelines for Future Agents

**Scope:** everything in this note is **inside the container only**. Never touch the host.
**Last updated:** 2026-10-07 15:25 UTC

---

## 1. What this container actually is (verified facts)

| Fact | Value (verified 2026-10-07) | How verified |
| --- | --- | --- |
| Base image | `python:3.10-slim` (Debian slim) | `.devcontainer/Dockerfile` |
| Python | 3.10.21 | `python --version` |
| Cores visible | 16 | `nproc` |
| RAM total | 3,858,620 kB ≈ **3.68 GB** | `/proc/meminfo` `MemTotal` |
| Swap | 1 GB | `/proc/meminfo` `SwapTotal` |
| cgroup v2 cpu.max | `max 100000` → **no CPU cap** | `/sys/fs/cgroup/cpu.max` |
| cgroup v2 memory.max | `max` (no cgroup cap; the 3.68 GB is the VM/WSL limit) | `/sys/fs/cgroup/memory.max` |
| Kernel | WSL2 `6.6.87.2-microsoft-standard-WSL2` | `uname -a` |

### ⚠️ Missing standard tools
`ps`, `top`, `htop`, `free`, `uptime`, `lsof`, `strace` are **NOT installed**. Only `df`, `nice`, `ionice`, `python` are present.
➡️ **Agents must not rely on `ps`/`top`/`free`.** Use `/proc` directly (helper below) or `python`. Do **not** run `apt-get install` just to inspect — the container is ephemeral and the build is frozen by `.devcontainer/Dockerfile`.

### ⚠️ The workspace is a Windows bind mount (biggest I/O latency vector)
```
D:\             270G  227G   43G  85%  /workspace     <-- Windows drive via WSL2 (slow I/O)
overlay         1007G  40G  916G   5%  /              <-- container root (fast, ephemeral)
```
`/workspace` is served from the **Windows `D:` drive** over WSL2 virtiofs. Every file read/write there — VS Code indexing, Pylance, `git` diff, Cline file reads, DuckDB on `/workspace/ICOCCO.duckdb` (116 MB) — pays a virtiofs penalty. This is the main *structural* cause of perceived slowness, and it **cannot be fixed from inside the container**. Prefer keeping hot scratch data on the container root (e.g. `/tmp`, the fast `overlay`) rather than `/workspace` when a task allows it.

---


## 2. Process map — what is normal, what to NEVER kill

There are ~29–32 processes. The important ones (verified):

| PID | What it is | Verdict |
| --- | --- | --- |
| 1 | `/bin/sh -c ... while sleep 1 & wait $!; do :; done` — **devcontainer keep-alive** | **DO NOT KILL.** Killing PID 1 stops the container. It respawns a `sleep 1` every second; that `sleep` shows as an *orphan with PPID=1* — this is **expected, not a leak**. |
| 233/263/188/284/309/385/1250/2033 | `node` = VS Code server + extension host + remote-cli helpers (`/vscode/vscode-server/...`) | **DO NOT KILL.** Core of the IDE. |
| 1405 | Pylance language server (`ms-python.vscode-pylance`) | **DO NOT KILL** (unless confirmed runaway). Normal steady CPU ~1–9%. |
| 773 | `pet server` (python-env-tools) | **DO NOT KILL.** |
| 337 | `copilot-runtime` (GitHub Copilot extension, ~40 threads) | **DO NOT KILL.** |
| 4574… | `/bin/bash --init-file .../shellIntegration` — VS Code integrated terminals | Leave running; these are user/agent shells. |
| `sleep` (transient, PPID=1) | devcontainer keep-alive child | **Ignore.** |

**Nothing in this container is a leftover agent process.** There are no zombies, no orphaned node/python runners, and no `sleep 240`-style injected delays (scanned all `/proc/*/cmdline`).

---

## 3. Canonical health-check routine (no `ps` needed)

**Recommended cadence:** run this **every ~10 tool calls**, and **immediately before/after any heavy command** (notebook execution, FoodBERT encode, K-means, big DuckDB query).

### 3a. One-liner snapshot (fast)
```bash
echo "time $(date +%H:%M:%S.%3N)"; cat /proc/loadavg; \
grep -E 'MemTotal|MemFree|MemAvailable' /proc/meminfo; \
echo "mem_current $(cat /sys/fs/cgroup/memory.current)"; \
echo "procs $(ls -d /proc/[0-9]* | wc -l)"; du -sk /tmp
```
Interpreting: `loadavg` on 16 cores — < 4 is fine; `MemAvailable` should stay > ~800 MB; `mem_current` should stay < ~3 GB (OOM risk above ~3.68 GB incl. page cache).

### 3b. Top-CPU sample (2 s, `/proc`-based — save as `/tmp/health_cpu.py`)
```python
import os, time
def rd(p):
    try:
        with open(p, 'rb') as f: return f.read()
    except Exception: return b''
def snap():
    d = {}
    for pid in os.listdir('/proc'):
        if not pid.isdigit(): continue
        s = rd(f'/proc/{pid}/stat').decode(errors='replace')
        if not s: continue
        try:
            rp = s.rindex(')'); rest = s[rp+2:].split()
            d[pid] = int(rest[11]) + int(rest[12])
        except Exception: pass
    return d, sum(int(x) for x in rd('/proc/stat').decode().splitlines()[0].split()[1:])
a, ta = snap(); time.sleep(2.0); b, tb = snap()
dt = max(tb - ta, 1)
out = []
for pid in b:
    if pid in a and b[pid] - a[pid] > 0:
        cmd = rd(f'/proc/{pid}/cmdline').decode(errors='replace').replace('\x00', ' ').strip()
        out.append(((b[pid]-a[pid])/dt*100, pid, cmd[:140]))
for c, pid, cmd in sorted(out, reverse=True)[:15]:
    print(f'{c:6.1f}%  {pid:>6}  {cmd}')
```
Note: CPU% here is **out of 16 cores**, so a value of `100%` = one full core. The sum of values ≈ whole-system busy %.

### 3c. Memory pressure scale (observed)
- Idle-with-IDE: `mem_current` ≈ **2.3–2.7 GB**, `MemAvailable` ≈ 0.9–1.3 GB → healthy.
- Under a real compute task: climbs toward 3 GB; a DuckDB/FoodBERT job can push toward the 3.68 GB wall → **risk of OOM-killer**. Watch `Dirty`/`writeback` in `/sys/fs/cgroup/memory.stat` during DuckDB writes.

## 3d. Slow-command / I/O latency diagnostic rule

**Rule:** During all work, agents should pay attention to the **execution time of individual commands/tool calls**, especially basic I/O and filesystem operations.

If a normally lightweight operation takes **>3 seconds** unexpectedly — for example:

* `ls`, `find`, `stat`, `du`, `cat`, `head`, `tail`
* reading a small file
* simple `grep`/text processing
* basic shell commands
* lightweight Python startup/imports
* simple filesystem metadata operations

then **do not immediately assume the command itself is expensive**.

### Diagnostic procedure

1. **Notice and record the elapsed time** of the slow command.

   * Agents should keep an approximate mental/logged awareness of command execution times throughout the task.
   * Repeated slow commands are more significant than a single isolated spike.

2. **If a basic operation exceeds ~3 s, inspect container health immediately**, using the `/proc`-based checks from §3a/§3b as appropriate:

   * `/proc/loadavg`
   * `MemAvailable`
   * cgroup `memory.current`
   * cgroup CPU statistics
   * optionally the 2-second `/proc` CPU sampler for suspicious cases.

3. **If the container appears healthy** — no significant CPU saturation, no severe memory pressure/OOM symptoms, and no obvious local resource contention — then for operations involving external resources, **network latency becomes a likely explanation**.

   * Do not misattribute network delays to CPU, RAM, Python, or filesystem performance without evidence.
   * Distinguish local filesystem latency from network-dependent operations where possible.

4. **For `/workspace` filesystem operations**, remember that this is a Windows/WSL2 bind mount and can inherently be much slower than `/tmp` or the container's overlay filesystem. A >3 s operation involving `/workspace` should therefore be interpreted in that context before blaming the network.

5. **Compare when useful:**

   * Repeat a lightweight equivalent operation against `/tmp` or another container-local path.
   * If `/tmp` is fast but `/workspace` is slow, suspect **virtiofs/Windows bind-mount I/O latency**.
   * If both are slow while the container is healthy, investigate **system-wide or external/network latency**.
   * If the container is under CPU/memory pressure, diagnose the resource bottleneck first.

### Important operational principle

> **A command taking >3 s is a diagnostic signal, not a diagnosis.**

Agents should correlate **command execution time + system health + operation type + filesystem location** before concluding why something is slow.

This rule applies continuously throughout the task, not only during scheduled health checks.

---


## 4. Clean-up checklist

### Before heavy work
- `mkdir -p /tmp/scratch && cd /tmp/scratch` — keep scratch **off** the Windows bind mount (`/workspace`) to avoid virtiofs latency.
- Ensure no previous DuckDB connection is open (DuckDB = single-writer): check for stale `.wal`/`.tmp` orphans; prefer the connect/disconnect cell pattern in every SQL notebook.

### After heavy work (Phase 03)
- Remove your own scratch: `rm -rf /tmp/scratch /tmp/health_cpu.py` and any `*.log`/`*.png` you created.
- Do **not** delete `/tmp/vscode-*.sock`, `/tmp/mcp-*`, `/tmp/pyright-<livepid>-*`, `/tmp/python-languageserver-cancellation/<current>`, `/tmp/devcontainers-<current-session>`, or `/tmp/*.js` server files — these belong to the live VS Code session.

### Known junk sources to sweep periodically (created by *previous agent runs*, not the OS)
Pattern seen at 2026-10-07: dozens of throwaway experiment scripts and logs left in `/tmp`:
`exp.py`, `geo_exp.py`, `map_exp*.py`, `trans_exp.py`, `truth_exp.py`, `protocol.py`, `sweep.py`, `patch*.py`, `argmax_table.py`, `country_cont.py`, `extract_img.py`, `fix_cfg.py`, `syntax_check.py`, `leak.py`, plus matching `*.log`, and `pdfimg/` (extracted PDF images). Plus **stale** language-server dirs whose owning PID is dead (`pyright-<deadpid>-*`, `mcp-<old>`, `devcontainers-<old-session>`, old `python-languageserver-cancellation/<hash>` subdirs).
➡️ Safe sweep = remove those throwaway `.py`/`.log`, `pdfimg/`, and **stale** LS dirs **after confirming no `/proc/*/fd` links them**. Keep notebook `.bak` backups if present — they are possible user data.

---

## 5. Execution risks found in this workspace

1. **Windows bind mount (`/workspace`)** — dominant latency risk; unfixable in-container. Mitigate by scratch-in-`/tmp`, minimal large-file I/O on `/workspace`.
2. **No CPU cap + 16 cores advertised, but 3.68 GB RAM** — a parallel job (e.g. scikit-learn `n_jobs=-1`, transformers) can spike all cores (Docker Desktop "1600%") while RAM is the real bottleneck → OOM. Cap parallelism (`n_jobs` small, `OMP_NUM_THREADS`, `MALLOC_ARENA_MAX=2`) for memory-heavy steps.
3. **DuckDB single-writer** on `/workspace/ICOCCO.duckdb` (116 MB, on the slow mount) — never open two writers; always pair connect/disconnect cells.
4. **Ephemeral `/tmp`** — wiped on restart; never store anything you need across restarts there.
5. **`>1600%` CPU spikes** reported by Docker Desktop include the **whole WSL2 VM / sibling containers**, not only this container. At inspection time this container was ~15% busy, so the spike was not an ongoing in-container leak.

---

## 6. Quick reference — inspection commands used (all safe, read-only)
```bash
date +'%Y-%m-%d %H:%M:%S.%3N'          # timestamp
nproc; cat /proc/loadavg; grep Mem /proc/meminfo
cat /sys/fs/cgroup/memory.current /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/cpu.max /sys/fs/cgroup/cpu.stat
ls -la /tmp; du -sk /tmp
for f in /proc/[0-9]*/cmdline; do tr '\0' ' ' < "$f"; echo; done   # process list w/o ps
for fd in /proc/[0-9]*/fd/*; do readlink "$fd"; done               # open files w/o lsof
```

## 7. Cline & Extension Host Self-Diagnostic Protocols (FOR AGENTS)

### 7a. The Extension Host Risk Profile
- **Primary Bottleneck:** `/workspace` resides on a Windows `D:` drive bind mount via WSL2 virtiofs. Heavy I/O or massive context payloads will saturate virtiofs / VS Code WebSocket, leading to WebView white-screen crashes and Extension Host process termination.
- **Node.js Memory Limit:** The Extension Host process running Cline has tight V8 memory caps. Dumping >1,000 lines of output into the chat context will trigger a V8 OOM crash.

### 7b. Agent Pre-Call Safeguards (MANDATORY BEFORE TOOL RUNS)
Before running commands or outputting data, agents MUST enforce:
1. **Output Truncation:** Never run commands that dump unpaginated logs or huge dataframes directly to stdout. Always pipe through `head -n 50`, `tail -n 50`, or Python limiters (e.g., `df.head(20)`).
2. **Scratchpad Isolation:** Write all intermediate scripts, temporary JSONs, and raw logs to `/tmp/scratch/` (overlay filesystem), NEVER to `/workspace/` or `/workspace/.cline/`.

### 7c. Agent Self-Health Routine for Cline & VS Code Host
Agents should run this inline logic or script whenever checking process state:

```bash
# 1. Check Extension Host Node process count & total memory usage without ps/top
for pid in /proc/[0-9]*; do
  cmd=$(cat "$pid/cmdline" 2>/dev/null | tr '\0' ' ')
  if echo "$cmd" | grep -qE "vscode-server|saoudrizwan.claude-dev"; then
    echo "$pid: $cmd" | cut -c1-120
  fi
done

# 2. Check workspace I/O pressure & disk stat on /workspace vs /tmp
echo "--- Storage Pressure ---"
df -h /workspace /tmp
```

### 7d. Emergency Protocol when UI Latency / Stuttering is Detected
If any basic filesystem command (`ls`, `stat`, `cat`) on `/workspace` takes **> 3 seconds**:
1. **STOP** sending long text responses or multiple consecutive file operations.
2. Force garbage collection or memory flush by reducing context length.
3. Inform the user: *"Virtiofs I/O latency detected (>3s). Diverting scratch operations to /tmp to prevent WebView freeze."*