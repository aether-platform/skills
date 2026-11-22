---
name: emdash-site-build
description: Build a website on the EmDash CMS running on Cloudflare Workers, move an existing site (for example WordPress) onto it, and switch the public domain over with a way back. Use when a new site must be scaffolded from an EmDash template and deployed, when the pages, posts and media of an old site must be imported and their old URLs kept working with redirects, when site code changes must go through a preview version before production, or when www and the apex domain must be cut over to the Worker (or rolled back to the old origin). Uses `npm create emdash`, the `emdash` CLI and `wrangler`.
---

# EmDash site build

This skill gives a fixed procedure for building a site with
[EmDash](https://github.com/emdash-cms/emdash) on Cloudflare Workers (D1 for
the database, R2 for media) and for moving a live site onto it without losing
pages, URLs or the way back. It uses the project scaffolder
(`npm create emdash@latest`), the `emdash` CLI that ships with the `emdash`
package, and Cloudflare's `wrangler`. It does not add any tools of its own.

The work runs in five phases. Each one ends at a point where you can stop,
show the result and ask for the next decision:

1. **A. Inventory** of the current site (read-only).
2. **B. Staging build** — scaffold, deploy, first admin.
3. **C. Content import** — media, pages, posts, links.
4. **D. Site code** — layout, routes, redirects, through preview versions.
5. **E. Cutover** — DNS and custom domains, with snapshots and rollback.

The checks in **F** are used by phases C–E against staging, preview versions
and production.

Run `<command> --help` for the exact options of the versions installed in your
site project (`npx emdash --help`, `npx wrangler --help`). For changing single
pages and posts on a site that is already live, use the `emdash-publishing`
skill in this repository instead.

## Ground rules

- **The old site stays up until the new one is proven.** Do not delete, disable
  or cancel the old site or its hosting during the build. It is the rollback
  target until the owner decides otherwise.
- **The public domain changes only on an explicit decision.** Phases A–D never
  touch the production DNS records. Phase E starts only when the owner says go,
  and every DNS change has a snapshot before and after.
- **Read-only towards the old site.** Inventory and import use public pages,
  public APIs (for WordPress, the REST API under `/wp-json/`) and exports the
  owner provides. Do not log in to the old CMS or change it unless asked.
- **Credentials stay out of files and messages.** Log in with `wrangler login`
  and `emdash login`, or pass tokens from the environment. Give an API token
  only the permissions the phase needs, limited to the one account, and record
  its name, not its value. Do not write account IDs, zone IDs, tokens or
  internal hostnames into the repository, commit messages, requests or
  reports that leave your team.
- **Every deploy is a version you can return to.** Upload a version, check it
  on its preview URL, then deploy it. Record the version that was live before
  (`wrangler deployments list`) so `wrangler rollback <version-id>` is ready.
- **Do not ship work you have not seen.** Before changing site code, confirm
  that your local source is what is currently deployed (no newer files than
  the live build, no other person's unfinished change).
- **Text is moved, not edited.** Imported pages keep every character of their
  source. Wording fixes belong to the content owner; write them down as open
  points.

## A. Inventory of the current site

Work in one folder per site and keep every file you fetch.

1. **DNS and hosting.** Record the name servers, the records for `www` and the
   apex, mail records (MX, SPF, DKIM, DMARC) and verification TXT records
   (`dig`, or an export from the DNS provider). Mail and verification records
   must survive the cutover unchanged.
2. **Content counts.** Count pages, posts, other content types and media from
   the public side (for WordPress: `/wp-json/wp/v2/pages`, `/posts`, `/media`,
   `/types`, reading the `X-WP-Total` header). Save the JSON of each item: it
   is the import source if no export is provided.
3. **What the public side cannot show.** Drafts, private items, private media,
   theme, plugins and forms are only visible in the old CMS's admin screen.
   List them as questions for the owner; do not guess.
4. **URL list.** Write the full list of public URLs (pages, posts, archives,
   feeds, sitemap, `robots.txt`) with their current status. This list is the
   test set for phases C–E.
5. **Things served by others.** Note paths on the same host that belong to
   other applications (for example Workers routes for a sub-path, or a
   different app under `/app/`). They must keep working after the cutover.
6. **Functions that will not move.** Forms, search, analytics tags,
   e-commerce and other plugin features do not come with the content. List
   each one and whether it is needed; a form that does not work today is
   "not present", not something to migrate.

## B. Staging build

### B1. Scaffold

EmDash and current `wrangler` need a recent Node.js (this procedure was run
with Node 22). If the system Node is older, install a separate Node into the
work folder and put it first on `PATH` for this project only, instead of
replacing the system one.

```bash
npm create emdash@latest <project> -- --template cloudflare:<template> --pm npm --install --yes
```

- `<template>` is one of the templates the scaffolder lists (`blog`,
  `starter`, `marketing`, `portfolio`). Pick the one whose page structure is
  closest to the site; the look is rebuilt in phase D anyway.
- The project contains `wrangler.jsonc` (Worker name, D1 database, R2 bucket),
  `astro.config.mjs` (the `emdash` integration with `d1()` and `r2()`) and a
  seed file referenced from `package.json`.
- Rename the Worker, the D1 database and the R2 bucket in `wrangler.jsonc` to
  names that mark them as this site's staging resources, before the first
  deploy. The first deploy creates them under those names.

### B2. First deploy

```bash
npx wrangler login            # or a scoped API token in the environment
npm run deploy                # astro build && wrangler deploy
```

- The first deploy provisions the D1 database and the R2 bucket named in
  `wrangler.jsonc` (and may add others, such as a KV namespace for sessions).
  List what was created and keep the list in the record.
- Check the site and the admin screen (`/_emdash/admin`) on the `workers.dev`
  URL. Do not attach a custom domain yet.
- Read the template's defaults you did not choose: cron triggers in
  `wrangler.jsonc`, sandboxed plugins (need Worker Loader on a paid plan),
  `npm audit` warnings. Record them as open points rather than changing them
  silently.

### B3. First administrator

- The admin setup wizard on `/_emdash/admin` creates the first administrator.
  **Until it is completed, anyone who finds the URL can run it.** Have the site
  owner complete it right after the first deploy, or restrict access to the
  staging host until then. Creating the owner's account or passkey is the
  owner's step, not the agent's.
- Then log the CLI in to the instance: `npx emdash login -u <staging URL>`
  (device authorisation; the owner approves it). Check with
  `npx emdash whoami`.

## C. Content import

### C1. Collections

1. Compare the content types of the old site with the template's collections
   (`npx emdash schema list`, `npx emdash schema get <collection>`).
2. Create what is missing, with the fields the import needs:

   ```bash
   npx emdash schema create posts --label "Posts"
   npx emdash schema add-field posts content --type portableText --label "Content"
   npx emdash schema add-field posts excerpt --type text
   ```

   Keep the old site's date and original URL as fields (for sorting and for
   building the redirect list).

### C2. Media first

1. Download the public media files into the work folder.
2. Upload each with its alt text and caption, and build a map from the old
   media ID / URL to the new media ID / URL:

   ```bash
   npx emdash media upload <file> --alt "<alt>" --caption "<caption>" --json
   ```

3. Expect differences and record them, do not hide them:
   - some file types may be refused (for example SVG — check the upload
     result); decide with the owner whether to convert them or serve them as
     static files from the site code;
   - files with identical content can be stored once, so the new media count
     can be lower than the number uploaded;
   - private media that the public API does not return are not imported.

### C3. Pages and posts

1. Convert each item's rendered HTML to Portable Text. For WordPress content
   the `@emdash-cms/gutenberg-to-portable-text` package (installed with
   EmDash) does this; keep the conversion script in the work folder.
2. While converting, rewrite image URLs to the uploaded media (C2) and
   internal links to the new slugs. Keep a list of links you could not
   resolve and say for each whether it was already broken on the old site.
3. Create the items and keep a map from old ID / URL to new slug:

   ```bash
   npx emdash content create pages --slug <slug> --file <item>.json --json
   ```

   Create them as drafts (`--draft`) if the owner wants to review before they
   appear; `npx emdash content publish <collection> <slug>` afterwards.

### C4. Check the import

- **Counts:** items per type, old vs new, and media uploaded vs stored, with
  every difference explained.
- **Text:** compare the text of each imported item with its source sentence
  by sentence. Report the number of items and whether all match.
- **Status:** every new URL returns 200; every image in the bodies returns 200
  from the new media store.
- **What did not move:** layout blocks that were flattened, plugin features,
  drafts and private items, SEO fields, categories and tags, authors and
  comments. This list goes to the owner with phase D.

## D. Site code: layout, routes and redirects

Content lives in the CMS; the look, the URL patterns and the redirects live in
the site code (Astro components, `astro.config.mjs`, `src/worker.ts`). Every
change here goes through the same loop.

### D1. Keep the old URLs working

1. From the URL list (A4) and the import map (C3), write one redirect per old
   URL that changes, in the `redirects` option of `astro.config.mjs`:

   ```js
   const legacyRedirects = {
     "/old/parent/page": { status: 301, destination: "/page" },
     "/2025/06/06/post-slug": { status: 301, destination: "/posts/post-slug" },
     "/feed": { status: 301, destination: "/rss.xml" },
   };
   export default defineConfig({ redirects: legacyRedirects, /* ... */ });
   ```

2. Cover what the old site answered with 200 besides pages: parent pages
   without content, category archives, feeds, the old sitemap. A URL that
   was 200 before and 404 after is a regression at cutover.
3. When a kind of page needs its own URL pattern (for example
   `/privacy/<name>/`), add a route for the pattern (a dynamic page such as
   `src/pages/privacy/[slug].astro`) rather than one route per page, so the
   next page of that kind is a CMS change only.

### D2. Static files

Files in `public/` are served as static assets. By default Workers static
assets may redirect `name.html` to `name`; a file that must be served under its
exact name with 200 (for example a search engine's ownership file) needs
`"assets": { "html_handling": "none" }` in `wrangler.jsonc`. Check that no
other HTML in `public/` depends on the default before changing it.

### D3. The change loop

1. Confirm the local source matches the live version (see Ground rules) and
   save copies of the files you will change.
2. Make the change; build locally (`npm run build`, `npm run typecheck`).
3. Upload a version without moving traffic, and check it on its preview URL:

   ```bash
   npx wrangler versions upload --message "<what changed>"
   ```

4. Run the checks in phase F against the preview URL.
5. Deploy that version to 100% and record the previous one:

   ```bash
   npx wrangler deployments list
   npx wrangler versions deploy <version-id>@100% --message "<what changed>"
   ```

6. Rollback: `npx wrangler rollback <previous-version-id>`, and restore the
   saved source files.

## E. Cutover

### E1. Gate before switching

All of these must hold, measured just before the switch:

- every URL in the staging sitemap returns 200;
- every URL of the old site (A4) reaches its new counterpart with 200,
  directly or through one 301;
- forms that must work have been submitted once on staging and arrived;
- a full DNS export taken now matches the last snapshot (nothing changed
  underneath you), together with the Workers routes and custom domains.

### E2. Switch `www`

1. Save the DNS records, Workers routes and custom domains (snapshot
   "before").
2. Remove the `www` record that points at the old origin and, right after,
   add a Workers **custom domain** for `www` on the site's Worker (dashboard,
   or the `routes` entry with `custom_domain: true` in `wrangler.jsonc`).
   Cloudflare then manages the DNS record for that name itself.
3. Check immediately and repeatedly for a minute: status 200, content from the
   new site, no trace of the old one (for WordPress, no `wp-content`).
4. Run the full checks (phase F) against the public domain.

### E3. Switch the apex

Decide first what the apex should do. If the old site redirected the apex to
`www`, keep that behaviour: other applications may be reachable only through
`www` (routes registered as `www.example.com/app*`), and moving the apex to the
Worker as a second site would turn those paths into 404s. A small handler in
the Worker entry (`src/worker.ts`) that answers requests for the apex host with
a 301 to the same path and query on `www` keeps it, after which the apex can
become a custom domain of the same Worker. Deploy that handler through the
change loop (D3) before switching the apex.

If a check fails after a switch, put that name back at once (remove the
custom domain, recreate the old record from the snapshot) and investigate on
staging.

### E4. After the switch

- Take the "after" snapshot and diff it with "before": only the records you
  meant to change may differ. Mail and verification records must be identical.
- Paths owned by other applications (A5) still answer from those applications.
  Workers routes are matched before custom domains, so a route for a sub-path
  keeps serving that path — check it, do not assume it.
- Leave the old origin running. Write down, as the decommission procedure for
  the owner: export the old CMS's data (database, uploaded files, private
  media); check the hosting's admin screen for other domains, mail and jobs;
  keep it for an agreed period as the rollback target; only then cancel it,
  and confirm afterwards that no DNS record points at it.

### E5. Rollback

- Code: `npx wrangler rollback <previous-version-id>`.
- Domain: remove the custom domain for the name, recreate its record pointing
  at the old origin (values from the "before" snapshot), then compare the full
  DNS export with that snapshot. Re-run the URL list against the old site.

## F. Checks (staging, preview and production)

Read-only, with `curl` and a browser:

- **Status:** every sitemap URL 200; every old URL reaches its counterpart;
  redirects are 301 to the intended destination, with and without the trailing
  slash; pages that should not exist are 404.
- **Page details:** `canonical` is the page's own public URL; `noindex` only
  where asked; analytics and verification tags present once.
- **Feeds and sitemap:** `/sitemap.xml` and the RSS feed list the published
  items and use the public host.
- **Text:** for imported and legal pages, the text inside `<main>` matches the
  source (C4).
- **Layout:** at a phone width (360px) and a desktop width (1280px), no
  horizontal overflow, images load (scroll to lazy-loaded ones), and take
  screenshots and look at them.
- **Mixed state after a switch:** for a few minutes, repeat requests to `www`,
  the apex and a few deep URLs; every answer comes from the new site.

## Report

Use this shape at the end of each phase:

```
Phase:       <A inventory / B staging / C import / D code / E cutover>
Site:        <public host>  staging: <workers.dev host>  (no IDs or tokens)
Resources:   <Worker, D1, R2, other bindings created>
Content:     pages <old→new>  posts <old→new>  media <uploaded/stored>  differences: <...>
URLs:        old URLs reaching a 200: <n/n>  sitemap 200: <n/n>  redirects added: <n>
Versions:    live <before> → <after>  (preview checked: <yes>)
DNS:         snapshot before/after: <files>  differences: <only the intended records>
Rollback:    <wrangler rollback ... / domain steps from E5>
Open points: <things not moved, owner decisions needed, defaults not changed>
```
