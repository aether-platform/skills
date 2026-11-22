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
