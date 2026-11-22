# skills

Agent Skills published by AetherPlatform.

Each skill lives in its own directory under [`skills/`](skills/) and consists
of a `SKILL.md` (instructions only, no code). Install only the skills you need.

## Skills

| Skill | Path | Use it when |
|-------|------|-------------|
| `zuul-skill` (Zuul failure triage) | [`skills/zuul-skill/`](skills/zuul-skill/SKILL.md) | A Zuul build, buildset, job or change **has failed**, is flaky or is stuck, and you need the cause. |
| `zuul-job-authoring` (Zuul job authoring) | [`skills/zuul-job-authoring/`](skills/zuul-job-authoring/SKILL.md) | You need to **add or change** a Zuul job, its parent/nodeset/vars/secrets, or a project's check/gate stanza, and confirm the new config loads and runs. |
| `emdash-publishing` (EmDash publishing) | [`skills/emdash-publishing/`](skills/emdash-publishing/SKILL.md) | A page (e.g. a privacy policy or terms of use) or a post on an EmDash site must be **added or replaced**, you need to **request** that from the site operator, or you need to **confirm** the change is live. |
| `emdash-site-build` (EmDash site build) | [`skills/emdash-site-build/`](skills/emdash-site-build/SKILL.md) | A **whole site** must be built on EmDash (Cloudflare Workers), an existing site **moved** onto it with its old URLs kept, or the public domain **cut over** to it (or rolled back). |
| `kubernetes-safe-change` (Kubernetes safe change) | [`skills/kubernetes-safe-change/`](skills/kubernetes-safe-change/SKILL.md) | You must **change** something on a Kubernetes cluster (manifests, Helm values, RBAC) or try it on a **throwaway** kind/k3d cluster, and need to be sure of the cluster, predict the rollout, validate, apply, verify and keep the rollback. |
| `oam-plane-separation` (OAM plane separation) | [`skills/oam-plane-separation/`](skills/oam-plane-separation/SKILL.md) | A network function's **service** and **OAM** (operations, administration and maintenance) traffic must run on separate interfaces and VRFs in a Kubernetes Pod (Multus + CNI `vrf`), and you need to **prove** they stay separate. |

## Zuul skills

### How to choose

- Something already ran and went wrong → `zuul-skill`.
- Nothing exists yet, or the job definition itself is what you are changing →
  `zuul-job-authoring`.
- They connect: `zuul-job-authoring` ends by checking config errors and the
  first build of the new job; if that build fails, continue with `zuul-skill`.

Both are read-only towards Zuul. `zuul-job-authoring` writes only to the
repository working tree; configuration reaches Zuul through your normal review
flow.

### Requirements

- An MCP client that supports Agent Skills (for example Claude Code).
- **[mcp-zuul](https://github.com/imatza-rh/mcp-zuul), installed and
  registered separately.** This repository does not include or redistribute
  it. Example for Claude Code:

  ```bash
  claude mcp add zuul \
    -e ZUUL_URL=https://zuul.example.com \
    -e ZUUL_DEFAULT_TENANT=example-tenant \
    -- uvx mcp-zuul
  ```

  `ZUUL_URL` and `ZUUL_DEFAULT_TENANT` above are placeholders. If your Zuul
  needs a token, pass `ZUUL_AUTH_TOKEN` from your environment; do not write it
  into config files. See the mcp-zuul README for all options.

- **Read-only operation is the default.** mcp-zuul starts with
  `ZUUL_READ_ONLY=true`, which hides the pipeline-changing tools. The skills
  assume this and do not call write tools without asking the user.

### Usage

Failure triage (`zuul-skill`):

- "Why did this Zuul build fail? <build URL>"
- "Change 12345 failed in check. Is it our change or infra?"
- "Is job `example-unit-tests` flaky?"

Job authoring (`zuul-job-authoring`):

- "Add a lint job for `example-org/example-project` to check and gate, based on
  what similar projects use."
- "Write a publish job that uses the registry secret; don't put any secret
  values in the change."
- "I pushed the new job. Did the config load and did it run?"

### Scope

- `zuul-skill` covers diagnosis and reporting.
- `zuul-job-authoring` covers drafting job/project configuration and playbook
  layout in a repository, and verifying it through MCP after it is proposed.
  It never writes secret values and does not push configuration to Zuul
  directly.
- Neither covers re-running, enqueueing or dequeueing changes.
- Tool names follow mcp-zuul. If mcp-zuul renames or removes a tool, update
  both `SKILL.md` files accordingly.

## EmDash skills

### How to choose

- The site does not exist yet, is being moved from another CMS, or its domain
  is being switched → `emdash-site-build`.
- The site is live and one page or post must be added, replaced or checked →
  `emdash-publishing`.
- They connect: once `emdash-site-build` has cut the domain over, later
  content changes follow `emdash-publishing`.

### emdash-publishing

`emdash-publishing` covers adding and replacing pages and posts on a site
built with the [EmDash](https://github.com/emdash-cms/emdash) CMS: how to
request a replacement from the site operator, how the operator publishes it
with the `emdash` CLI (backups, revision tokens, draft → publish, images,
rollback), and how both sides check the public site afterwards.

### Requirements

- Operator side: the site project with the `emdash` package installed (the CLI
  is `node_modules/.bin/emdash`), and an account on the EmDash instance
  (`emdash login`, or a token passed from the environment). The skill never
  writes credentials into files.
- Requester side: no CMS access is needed; `curl` and a browser are enough for
  the checks.

### Usage

- "Replace the privacy policy and terms pages with the version at commit
  `<sha>` of `<repo>`, and confirm they are live."
- "Write the request to the site operator for the new terms page, with the
  checks we will use afterwards."
- "Publish this news post with the two screenshots."

### emdash-site-build

`emdash-site-build` covers building a site with EmDash on Cloudflare Workers
and moving a live site onto it: a read-only inventory of the old site, the
staging build (`npm create emdash`, first deploy, first administrator), the
content import (collections, media, pages and posts with the `emdash` CLI),
site code changes through preview versions (`wrangler versions upload` →
check → `wrangler versions deploy`), redirects for the old URLs, and the
cutover of `www` and the apex with DNS snapshots and rollback.

Requirements:

- A Cloudflare account with Workers, D1 and R2, and `wrangler` (installed with
  the project). Log in with `wrangler login` or a scoped API token from the
  environment.
- A recent Node.js for the project (the procedure was run with Node 22).
- Read access to the old site's public pages and API; admin access to the old
  CMS only if the owner provides it. The old site is never changed.

Usage:

- "Inventory our WordPress site and build an EmDash staging copy of it."
- "Import the pages, posts and media and list everything that did not move."
- "Add redirects for all old URLs, check them on a preview version, then
  deploy."
- "Cut www over to the Worker, keep the apex redirecting to www, and give me
  the rollback steps."

## Kubernetes skill

`kubernetes-safe-change` covers changing a Kubernetes cluster as an agent:
preparing a task-only kubeconfig and proving which cluster it reaches,
read-only inspection (how the workload was deployed, StorageClass defaults,
existing failures), predicting the effect by rendering and diffing (which Pod
templates change; why a ConfigMap change alone does not restart Pods),
server-side dry-run and admission, applying with `kubectl` / kustomize /
`helm` and watching the rollout, rollback, least-privilege ServiceAccounts
proven with `kubectl auth can-i --as`, and disposable kind/k3d clusters with
stop conditions and clean teardown.

### Requirements

- `kubectl` (kustomize is built in) and, for Helm releases, `helm`. For
  throwaway clusters, `kind` or `k3d` and a container runtime with free disk
  space.
- A kubeconfig for the target cluster, provided for the task. The skill keeps
  it in the work folder and never writes to `~/.kube/config`.
- Writes (`apply`, `upgrade`, `rollback`, `delete`) only with the owner's
  approval for that cluster and change.

### Usage

- "Which cluster does this kubeconfig reach, and is it the dev cluster the
  infra repo describes? Read-only."
- "Will this values change restart the scheduler? Render it and tell me which
  Pods roll."
- "Apply this kustomization to the staging namespace, watch the rollout and
  give me the rollback command."
- "Give the CI job a ServiceAccount that can only patch this Deployment, and
  prove it cannot exec into Pods or read Secrets."

## OAM skill

`oam-plane-separation` covers splitting a network function's traffic into a
platform plane (`eth0`), a service plane and an OAM plane inside Kubernetes
Pods: Multus, the CNI `macvlan` and `vrf` plugins and whereabouts, one
NetworkAttachmentDefinition per interface, per-interface socket binding,
the isolation checks (interface→VRF, route leaks, reachability including
ICMP, captures, per-process socket ownership), a negative control without
the `vrf` plugin, and fault cases.

### How it relates to the Kubernetes skill

- `kubernetes-safe-change` is the general procedure for touching a cluster
  (context, dry-run, apply, rollout, throwaway clusters).
- `oam-plane-separation` is the network design and the isolation checks on
  top of it. Use both when you build the lab or change a NAD.

### Requirements

- A cluster where you may install CNI plugins on the nodes and deploy Multus
  and whereabouts (a throwaway kind/k3d cluster works for the lab form).
- An observation container or `kubectl debug` access to the function's
  network namespace with `ip`, `ss` and `tcpdump`. The observation container
  needs privileges the function itself must not have; keep them separate.

### Usage

- "Design the NADs and socket bindings so the admin API is only on the OAM
  VRF and client and upstream traffic stay on the service VRF."
- "Run the isolation checks on this Pod and the `-novrf` control, and tell me
  which checks actually detect a missing VRF."
- "Take the upstream interface down and show that the function fails closed
  without leaking to OAM or `eth0`."

## Install

Point your client at `skills/<name>`: copy (or symlink) that directory into a
skill directory your client reads. For Claude Code, user-level skills live in
`~/.claude/skills/<name>/` and project-level skills in `.claude/skills/<name>/`.

```bash
git clone https://github.com/aether-platform/skills.git
cd skills

# one skill
cp -r skills/zuul-skill ~/.claude/skills/

# or several
cp -r skills/zuul-skill skills/zuul-job-authoring ~/.claude/skills/
```

The directory name matches the `name` in each skill's frontmatter; keep it
unchanged when copying.

## History

The two Zuul skills were previously published in
[aether-platform/zuul-skill](https://github.com/aether-platform/zuul-skill).
They moved here unchanged (same names, same content); only their location
changed.

## License

This repository is licensed under the [MIT License](LICENSE).

mcp-zuul is a separate project by Itay Matza, licensed under the
[Apache License 2.0](https://github.com/imatza-rh/mcp-zuul/blob/main/LICENSE).
It is referenced here as a dependency only; no code from it is included.
