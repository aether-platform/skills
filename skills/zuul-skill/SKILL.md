---
name: zuul-skill
description: Triage failed Zuul CI builds through the mcp-zuul MCP server. Use when a Zuul build, buildset, job or change has failed, is flaky, or is stuck in a pipeline and you need to find the failing task, read the relevant part of the build log, decide whether the cause is the job definition, the node/infrastructure, or an external dependency, and report it. Read-only by default.
---

# Zuul build failure triage

This skill gives a fixed procedure for investigating Zuul CI failures with the
tools of [mcp-zuul](https://github.com/imatza-rh/mcp-zuul). It does not add any
tools of its own.

Tool names below are the names exposed by mcp-zuul. Your client may show them
with a prefix (for example `mcp__zuul__get_build_failures` in Claude Code when the
server is registered as `zuul`).

## Ground rules

- **Read-only.** Do not call `enqueue`, `dequeue`, `promote` or
  `reenqueue_buildset`. They are hidden unless the server runs with
  `ZUUL_READ_ONLY=false`; even if they are visible, ask the user first.
- `autohold_create` / `autohold_delete` are available even in read-only mode.
  They hold or release real nodes, so only call them when the user asks.
- Do not paste tokens, internal hostnames or full log URLs from a private Zuul
  into places outside the conversation (issues, chat, commits) unless the user
  asks for it.
- Keep log reads narrow. Prefer structured output and grep/range reads over
  loading whole logs.

## Procedure

### 1. Identify the failed build

Start from what the user gave you:

| Input | Tool |
|-------|------|
| Build URL or build UUID | use it directly in the next step (`url` or `uuid`) |
| Change / PR number | `get_change_status` (live jobs, or the latest completed buildset) |
| Buildset URL or UUID | `get_buildset` |
| Project / job name only | `list_builds` with `project`, `job_name`, `result` (e.g. `FAILURE`), small `limit` |

If the build is still running, use `stream_build_console` (optional extra) or
wait; the steps below assume a completed build.

If no build ran at all, go to step 4 ("job did not run") instead.

### 2. Get structured failure data

- `get_build_failures` — failed play, task, host, `msg`, `rc`, stderr/stdout
  from `job-output.json`. This is the primary source.
- `diagnose_build` — the same structured data plus the matching log context
  in one call. Use `brief=true` when you only need the classification.
- `get_build_test_results` — when the job produces JUnit XML, to get the
  failing test cases instead of reading test output by hand.

Write down: job name, result, failed task, host, return code, first error
message.

### 3. Read only the log you need

- `tail_build_log` — last N lines of the run phase (default 50, max 500).
  Fastest check when step 2 was not conclusive.
- `get_build_log` with `grep` (regex) and `context` — find specific errors.
- `get_build_log` with `start_line` / `end_line` — read an exact range around a
  hit.
- `get_build_log` with `log_name` or `browse_build_logs` — other files in the
  log directory (inventory, artifacts, service logs).
- `get_build_anomalies` — only if the server has `LOGJUICER_URL` configured.

Reuse the log URL returned by earlier calls (`direct_log_url`) to avoid
refetching build metadata.

### 4. Isolate the cause

Classify the failure into one of three buckets and collect evidence for it.

**A. Job definition / change content**
- `get_freeze_job` — the resolved job after inheritance (nodeset, playbooks,
  timeout; `include_vars=true` only when variables matter).
- `get_job`, `get_project`, `get_freeze_jobs` — which jobs run and where they
  come from.
- `get_config_errors` — when the job did not run or the config is broken.
- Compare with the last passing build of the same job (`list_builds` with
  `result=SUCCESS`) to see what changed.

**B. Node / infrastructure**
- Signs: `NODE_FAILURE`, `RETRY_LIMIT`, `TIMED_OUT`, unreachable hosts,
  disk/memory exhaustion, failures in pre-run.
- `list_nodes`, `list_labels`, `list_providers`, `list_images` — node supply.
- `list_semaphores` — jobs waiting on locks.
- `get_components`, `list_system_events` — scheduler/executor state and
  reconfigurations.
- `get_build_times` / `get_job_durations` — timeouts that track a duration
  trend.

**C. External dependency**
- Signs: package mirror / registry errors, DNS or TLS errors to third-party
  hosts, upstream API rate limits, a dependent change or repo.
- `find_flaky_jobs` — a high, mixed failure rate points to an intermittent
  external cause rather than the change under test.
- Check whether other projects using the same dependency fail at the same
  time (`list_builds` across projects).

If the evidence does not fit one bucket, say so and list what is missing.

### 5. Report

Use this shape:

```
Build:      <job> on <project> (<pipeline>) — <result>
Reference:  <build uuid or change number>
Failed at:  <play / task / host>, rc=<n>
Error:      <one or two log lines, quoted>
Cause:      A job definition | B node/infra | C external dependency | unknown
Evidence:   <what you checked and what it showed>
Flaky?:     <find_flaky_jobs result, or "not checked">
Next step:  <recheck / fix in change / fix in job config / ask infra / hold node>
```

Keep the quoted log short. State what you did not check.
