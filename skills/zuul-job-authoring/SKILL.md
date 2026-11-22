---
name: zuul-job-authoring
description: Draft new Zuul CI jobs and project pipeline configuration that match an existing tenant, using the mcp-zuul MCP server to read what is already there. Use when you need to add or change a Zuul job definition, choose its parent and nodeset, wire it into check/gate for a project, lay out its playbooks, or confirm after proposing the change that the new config loads and the job runs. Reads Zuul through MCP; writes only to the repository working tree, never to Zuul.
---

# Zuul job authoring

This skill gives a fixed procedure for writing Zuul job configuration that fits
the tenant it will run in. It uses the read tools of
[mcp-zuul](https://github.com/imatza-rh/mcp-zuul) to look at existing jobs,
pipelines and projects, then drafts YAML and playbooks in the repository you
are working in. It does not add any tools of its own.

Tool names below are the names exposed by mcp-zuul. Your client may show them
with a prefix (for example `mcp__zuul__get_job` in Claude Code when the server
is registered as `zuul`).

For a build that fails after you add the job, switch to the `zuul-skill`
(failure triage) skill in this repository.

## Ground rules

- **Zuul is read-only from this skill.** Configuration changes go through a
  normal code review change (PR / MR / Gerrit change) in the repository. Do not
  call `enqueue`, `dequeue`, `promote`, `reenqueue_buildset`, `autohold_create`
  or `autohold_delete` unless the user asks.
- **Secrets: names and locations only.** Never write a secret value, token,
  password or private key into YAML, playbooks, commit messages or chat. Refer
  to a secret by its `name`, say which project defines it, and leave the
  encrypted payload to the user (they run `zuul-client encrypt` or their
  site's equivalent). mcp-zuul has no tool that reads or lists secrets.
- **Copy the tenant, not the internet.** Parent jobs, nodesets, labels and
  pipeline names differ per site. Use what the MCP tools show for this tenant;
  if something cannot be confirmed through them, mark it as an assumption in
  the draft and ask.
- **No internal values in shared places.** Do not paste internal hostnames,
  full log URLs or tenant-specific names into public repositories, issues or
  examples unless the user asks for it.

## What the MCP tools can and cannot show

| Need | Tool | Notes |
|------|------|-------|
| Tenants / capabilities | `list_tenants`, `get_tenant_info` | |
| Source connections | `get_connections` | Gerrit / GitHub / GitLab and their names |
| Pipelines | `list_pipelines` | Names and trigger types (e.g. which one is the gate) |
| Projects | `list_projects`, `get_project` | `get_project` shows each pipeline's job list for the project |
| Jobs | `list_jobs` (with `filter`), `get_job` | `get_job` shows parent, nodeset, timeout, branches, all variants |
| Resolved job | `get_freeze_job` | Final nodeset, timeout, playbooks after inheritance; `include_vars=true` only when vars matter |
| Job graph | `get_freeze_jobs` | Which jobs run for a pipeline/project/branch, with dependencies |
| Node types | `list_labels` | Labels that nodesets can request |
| Locks | `list_semaphores` | Existing semaphores a job may reference |
| Config errors | `get_config_errors` | Tenant-wide or filtered by `project` |
| After proposing | `get_change_status`, `list_buildsets`, `list_builds`, `get_build_failures`, `diagnose_build` | Verification (step 6) |

Not available through mcp-zuul: listing nodeset definitions directly (read
them from `get_job` / `get_freeze_job` of an existing job, or from the config
repository), reading secrets, and reading arbitrary repository files. Read
repository files (`zuul.yaml`, `.zuul.yaml`, `zuul.d/`, `.zuul.d/`,
`playbooks/`) from the local checkout instead.

## Procedure

### 1. Pin down the request

Write down before drafting:

- Target project (as Zuul names it — confirm with `list_projects` /
  `get_project`) and branch.
- What the job must do (unit tests, lint, build an image, deploy, …).
- Which pipelines it belongs in (usually check, and gate if it should block
  merges), and whether it should vote.
- Whether it needs credentials (→ secret, see step 3) or extra repositories
  (→ `required-projects`).

### 2. Survey what already exists

1. `get_project` for the target project: which jobs already run in which
   pipeline, and whether the project uses a project template.
2. `list_jobs` with a `filter` close to what you want (`lint`, `unit`, `tox`,
   `build`, the language name, the project's prefix). Prefer reusing or
   inheriting an existing job over writing a new one from `base`.
3. `get_job` on the closest candidates: note `parent`, `nodeset`, `timeout`,
   branch variants, and whether the job is `abstract` or `final`.
4. `get_freeze_job` for one candidate in the pipeline you will use, to see the
   resolved nodeset and the full pre-run / run / post-run playbook chain you
   would inherit.
5. `list_pipelines` to get the exact pipeline names; do not assume `check` /
   `gate` exist under those names.
6. `list_labels` if you need a nodeset that no existing job uses.
7. `get_config_errors` with `project` — fix or at least know about existing
   errors before adding more config.

Record the findings as a short table (candidate job → parent, nodeset,
timeout, playbooks). The draft should be traceable to it.

### 3. Design the job

**Parent (inheritance)**
- Choose the most specific existing job that already sets up what you need
  (repos, language toolchain, log collection). Only fall back to `base` (or
  the site's base) when nothing closer fits.
- Do not inherit from a job marked `final`. An `abstract` parent is fine; your
  job must then be concrete (define `run`, or inherit one).
- Inherited `pre-run` / `post-run` playbooks always run around yours; check the
  frozen chain from step 2 so you do not repeat setup.

**Nodeset**
- Reuse the nodeset of a similar job (by name) when possible.
- If you define one inline, use only labels returned by `list_labels`. A wrong
  label is not a config error; it shows up later as `NODE_FAILURE`.
- Omit `nodeset` if the parent's nodeset is already right.

**Vars**
- Put defaults in `vars` on the job; let project stanzas override per project
  only when needed.
- Keep host-specific settings in `host-vars` / `group-vars`.
- Do not put credentials in `vars`.

**Secrets**
- Reference by name in the job: `secrets: [<secret-name>]` (or the mapping form
  with `name` / `secret` when the playbook expects a different variable name).
- State where the secret is defined (which project, which file) and that the
  value is an encrypted blob the user supplies. Use a placeholder such as
  `<encrypted-by-user>`; never invent ciphertext.
- Expect restrictions: a job that uses a secret from an untrusted project is
  treated as post-review and will not run in a pre-review pipeline like check.
  Plan check/gate placement accordingly and confirm against your Zuul
  version's documentation.

**Other attributes worth deciding explicitly**
- `timeout` — base it on similar jobs (`get_job_durations` /
  `get_build_times` for those jobs give real numbers).
- `files` / `irrelevant-files` — to skip the job for unrelated changes.
- `required-projects` — only projects that exist in the tenant
  (`list_projects`).
- `voting: false` for a new job while it is being stabilised.
- `semaphores` — only names returned by `list_semaphores`.

### 4. Draft the configuration

Follow the layout the repository already uses (`zuul.yaml`, `.zuul.yaml`,
`zuul.d/*.yaml` or `.zuul.d/*.yaml`; do not introduce a second style). All
names below are placeholders.

```yaml
# .zuul.d/jobs.yaml
- job:
    name: example-project-unit
    parent: example-base-unit        # existing job found in step 2
    description: Run unit tests for example-project.
    # nodeset: inherited from parent (confirmed with get_freeze_job)
    timeout: 1800
    vars:
      example_test_target: unit
    run: playbooks/unit/run.yaml
    irrelevant-files:
      - ^docs/.*$
      - ^.*\.md$

- job:
    name: example-project-publish
    parent: example-base-publish
    description: Publish artifacts for example-project.
    secrets:
      - name: example_registry_credentials   # name only; value is supplied by the user
        secret: example-registry-secret      # defined in <project>/<file>
    run: playbooks/publish/run.yaml
```

```yaml
# .zuul.d/project.yaml
- project:
    check:
      jobs:
        - example-project-unit
    gate:
      jobs:
        - example-project-unit
    # post / promote / release pipelines only if they exist (list_pipelines)
```

Playbook layout:

```
playbooks/
  unit/
    run.yaml        # the job's own work
    pre.yaml        # only if the parent's pre-run is not enough
    post.yaml       # extra log/artifact collection, if needed
roles/              # project-local roles, if any
```

Points to check in the draft:

- Every job added to the gate is also in check, so it is not first seen
  failing in the gate.
- If the project uses a project template, add jobs consistently with it
  instead of duplicating the template's jobs.
- Job names follow the naming pattern seen in `list_jobs` for this tenant.
- Playbook paths in `run` / `pre-run` / `post-run` exist in the same
  repository as the job definition.

### 5. Check for common config errors before proposing

| Error | Typical cause | How to avoid |
|-------|---------------|--------------|
| Job not found / unknown parent | Typo, or the job lives in a repo not loaded for this tenant | Confirm every name with `list_jobs` / `get_job` |
| Cannot inherit from final job | Parent is `final` | Check `get_job` output |
| Abstract job cannot be run | Project stanza lists an abstract job | Use a concrete child |
| Job already defined / not permitted to shadow | Same name defined in another project | Pick a unique name (`list_jobs`) |
| Unknown nodeset / semaphore | Name not defined in the tenant | Reuse names seen via `get_job` / `list_semaphores` |
| Unknown project in `required-projects` | Project not in the tenant | Confirm with `list_projects` |
| Unknown pipeline | Pipeline name differs at this site | Use `list_pipelines` names |
| YAML / schema error | Indentation, list vs mapping, unknown attribute | Lint the YAML locally before pushing |
| Playbook not found (at run time) | Wrong path in `run` | Paths are relative to the repo root |
| `NODE_FAILURE` (at run time) | Label not provided | Use labels from `list_labels` |

### 6. Propose and verify through MCP

1. Push the change for review in the repository's normal flow. Zuul tests
   configuration changes to untrusted projects speculatively, so the change
   itself runs with the new job. Changes to config projects only take effect
   after merge; say so to the user.
2. `get_config_errors` with `project` — the new config must add no errors.
   Errors caused by the change are usually also reported back on the change.
3. `get_change_status` for the change — the new job should appear in the
   check pipeline's job list.
4. `get_freeze_jobs` for the pipeline/project/branch — the job is in the graph
   with the expected dependencies. `get_freeze_job` for the new job — resolved
   parent chain, nodeset, timeout and playbooks are what you designed.
5. When the build finishes, `list_builds` with `job_name` (and `change`) to
   find it. On failure, use `get_build_failures` / `diagnose_build`, or hand
   off to the `zuul-skill` triage procedure.
6. If the job never started, check `get_config_errors` again and the job's
   `files` / `irrelevant-files` and branch matchers.

### 7. Report

Use this shape:

```
Project:     <project> (<branch>)
Added:       <job names>  parent=<parent>  nodeset=<name or inherited>
Pipelines:   check=<yes/no>  gate=<yes/no>  voting=<yes/no>
Files:       <config files and playbooks added or changed>
Secrets:     <secret names and where they are defined, or "none">
Based on:    <existing jobs used as reference (step 2)>
Verified:    config errors=<none/...>  freeze-job=<ok/...>  first build=<result or "not run yet">
Assumptions: <anything not confirmable through MCP>
Next step:   <user encrypts secret / review / make voting / add to gate>
```
