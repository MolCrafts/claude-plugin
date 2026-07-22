---
name: jobs
description: "molq/molmcp read dashboard. MUST load (or call molq MCP tools) when the user or the plan needs: 查作业/作业状态/看日志/队列/squeue/qstat/bjobs/list jobs/job status/check the queue/pull squeue/is my job running/where can I submit — do NOT raw-shell squeue/scancel when molmcp molq tools exist. Modes list|get|logs|destinations|queue. Read-only; never submit/cancel."
argument-hint: "[list|get|logs|destinations|queue] [job_id|options…]"
---

> **Codex:** Read `../CODEX.md` before executing this shared workflow. Claude Code follows the workflow directly.

# /molq:jobs — Job Dashboard (read-only)

Drive **molq** job lifecycle **read** tools exposed by molmcp. Does **not** submit or cancel (use `/molq:submit`, `/molq:cancel`).

## Free-form auto (tier A/B)

When conversation implies a queue/dashboard need — including the **agent deciding** "I should check squeue / job status / logs" mid-task — load this skill **or** call the molmcp molq tools below directly. Prefer MCP over `ssh … squeue` / bare `squeue`.

Slash `/molq:jobs` is optional.

## MCP tools

Namespace `molq` (often `molmcp__molq_<name>` or `molq_<name>` — discover with the MCP tool list if unsure):

| Tool | Use |
|---|---|
| `list_jobs` | Store dashboard |
| `get_job` | One job (+ optional scheduler refresh / transitions) |
| `job_logs` | stdout/stderr text (tail; no follow) |
| `list_destinations` | Profiles + SSH Host aliases |
| `list_queue` | Live `squeue`/`qstat`/`bjobs` snapshot |

DB path: constructor / `MOLQ_DB_PATH` / molq default (`~/.molq/…`). Skills do not invent a second store.

## Procedure

### 0. Preflight

- Confirm molmcp molq tools are available. Missing → tell user to enable molmcp with molq provider and `pip install molcrafts-molq`.
- Parse `$ARGUMENTS` into a **mode** (default `list`):

| Mode | Args |
|---|---|
| `list` | optional `cluster=<name>`, `terminal` (include terminal states), `limit=N` |
| `get` | required `job_id`; optional `no-refresh`, `no-transitions` |
| `logs` | required `job_id`; optional `stream=stdout\|stderr\|both`, `tail=N` |
| `destinations` | optional `no-ssh` |
| `queue` | optional `scheduler=local\|slurm\|pbs\|lsf`, `cluster=`, `profile=`, `user=` |

Bare UUID-looking token → mode `get`. Phrases like 日志/logs → `logs` if id present.

### 1. Call tools

**list** → `list_jobs(cluster_name?, include_terminal?, limit?)`  
Default: active-only (`include_terminal=false`). User asks for history/done → `include_terminal=true`.

**get** → `get_job(job_id, refresh=true, include_transitions=true)` unless flags say otherwise.

**logs** → `job_logs(job_id, stream, tail)`. Default stream `stdout`, tail `200`. Report `missing: true` streams honestly.

**destinations** → `list_destinations(include_ssh=true)` unless `no-ssh`.

**queue** → `list_queue(scheduler, cluster?, profile?, user?)`. Local scheduler often returns `[]` — say so, not an error.

### 2. Report

Human-readable table or short cards: job_id, state, cluster, command (if present), paths. One-line F2:

```
/molq:jobs <mode>: <N rows | job_id state | …>
```

## Guardrails

- **Read-only.** Never call `submit_job` / `cancel_job` from this skill.
- Prefer molmcp over raw `squeue`/`qstat`/`bjobs` when tools are connected.
- Never invent job ids or log text; tool miss → report miss.
- Agent self-triggers OK: if you need queue truth to continue a plan, call `list_queue` / `list_jobs` without waiting for a slash command.
