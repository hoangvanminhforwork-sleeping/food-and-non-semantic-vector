# AGENT MISSION: System Inspection & Performance Optimization

## Context & Environment
- **Environment**: Linux container managed via Docker Desktop inside VS Code (Cline extension).
- **Environment Specs**: Read `.devcontainer/` folder for base setup, dockerfiles, and container configurations.
- **Scope Limit**: **STRICTLY CONTAINED WITHIN THE CONTAINER**. Do **NOT** run any commands attempting to access, modify, or interact with the host system.

---

## Reported Symptoms
1. **Terminal Output Latency**: Commands complete execution in the terminal, but output retrieval/insertion into Cline context takes unusually long.
2. **Slow Text Generation**: AI response generation rate is extremely slow.
3. **Execution Delay**: Tasks experience massive overhead due to container-agent harness and extension communication.
4. **CPU Spikes**: Container CPU utilization exceeds 100% capacity (e.g., Docker Desktop reports >1600% on a 16-core allocation).

---

## User Diagnostics & Initial Hypotheses
1. **Orphaned Processes**: Frequent abrupt task cancellations/kills by user (stopping agent mid-execution) may have left runaway child processes, stale node/python processes, or unkilled subshells.
2. **Blocking / Long-running Commands**: Previous agents might have injected long sleeps, lingering subshells, or background hanging tasks (e.g., `sleep 240`, background file watchers).
3. **Junk / Temporary Files**: Uncleaned temporary files or lockfiles in `/tmp` or project root due to improper VS Code/container shutdowns.

---

## Core Mission Objectives

### 1. Investigation & Diagnostics
- **Inspect Running Processes**: Analyze current processes inside the container (`ps aux --sort=-%cpu`, `top`, or `htop`). Identify orphaned Node.js, Python, language servers, or lingering subshell processes consuming high CPU/RAM.
- **Inspect `/tmp` and Temp Directories**: Identify accumulation of temporary files, socket leaks, uncleaned lockfiles, or large log files left by previous agent runs.
- **Identify Risk Vectors**: Read relevant configuration files, scripts, or agent execution traces to ensure no prolonged execution routines or command locks are running.

### 2. Remediation & Fixes
- Kill all identified zombie, orphaned, or unnecessary background processes inside the container.
- Clean up bloat/temporary files in `/tmp` or other temp cache locations safely.
- Apply immediate performance tuning or process cleanup within the container scope.

### 3. Documentation & Handover
- **Handover Note (`system_inspection/system_note.md`)**: Write operational guidelines for future agents, including:
  - Common process leaks and lingering background tasks to watch out for.
  - Recommended clean-up steps before and after running heavy commands.
  - **Self-Health Monitoring Routine**: Suggest or establish a standard practice (e.g., checking system resources and running processes every ~10 tool calls) to maintain system stability.
  - Potential execution risks found in this workspace.
- **Mission Execution Report (`system_inspection/mission_report.md`)**: Document findings, actions taken, killed processes, freed resources, and before/after latency metrics.

---

## Execution Guidelines & Mandatory Rules

### Rule 1: Time Tracking & Latency Measurement
Every single tool call (command execution, file read, file write) **MUST** incorporate a time check to measure execution duration and system response latency.
- Include timestamp execution for terminal commands:
  ```bash
  date +'%Y-%m-%d %H:%M:%S.%3N' && <COMMAND> && date +'%Y-%m-%d %H:%M:%S.%3N'
  ```
- Track and record elapsed time to identify whether latency originates from process overhead, I/O bottlenecks, or harness communications.

### Rule 2: Periodic System Health Check
- **Periodic Checks**: Every ~10 tool calls, perform a quick background process and memory check (`ps aux | grep -v root`, `free -m`) to ensure no newly spawned tasks are hogging system resources.

### Rule 3: Optimization & Batch Operations
- **Batch Operations**: Group file reads and terminal execution commands wherever possible to minimize tool-call roundtrips between Cline and VS Code.
- **Non-Destructive Safety**: Do not kill critical VS Code Server internal processes (`vscode-server`, active language servers required for workspace operation) unless explicitly confirmed as runaway/zombie instances.

---

## Checklist & Deliverables
- [ ] Read environment configurations in `.devcontainer/`.
- [ ] Check active processes (`ps aux --sort=-%cpu`) with timestamp logging.
- [ ] Inspect and clear bloat/junk in `/tmp/` and temporary directories.
- [ ] Terminate orphaned, hanging, or zombie processes (e.g., lingering `sleep`, orphan node/python runners).
- [ ] Create `system_inspection/system_note.md` (guidelines & health-check rules for subsequent agents).
- [ ] Create `system_inspection/mission_report.md` (detailed investigation findings, actions taken, and system metrics).