---
name: emdash-publishing
description: Publish and replace pages and posts on a site run by the EmDash CMS, using the `emdash` CLI, and request or verify such changes when another person or agent operates the site. Use when a legal page (privacy policy, terms of use) must be added or replaced with a new version from a repository, when a news post or its images must be published or updated, or when you asked the site operator for a replacement and need to confirm it is live. Covers the request, the CMS steps with backups, and the checks on the public site.
---

# EmDash publishing

This skill gives a fixed procedure for putting content on a site whose pages
are managed in [EmDash](https://github.com/emdash-cms/emdash) and for checking
that it went live. It uses the `emdash` command-line interface that ships with
the `emdash` package. It does not add any tools of its own.

It has two sides, and you may only be on one of them:

- **Requester** — you own the text (for example the app team that maintains a
  privacy policy in its repository) but do not operate the site. Follow
  "A. Requesting a change" and "C. Verifying on the public site".
- **Operator** — you have access to the EmDash instance. Follow "B. Publishing
  with the emdash CLI" and "C. Verifying on the public site".

Command names below are those of the `emdash` CLI (`emdash --help`,
`emdash content --help`, `emdash media --help`). Run `emdash <command> --help`
for the exact options of the version installed in your site project.

## Ground rules

- **Do not change the text you were given.** A legal page is published with
  the wording of its source exactly, including notices such as "draft before
  legal review". Removing such notices, changing dates or fixing wording is the
  text owner's job, not the publisher's. Write down anything that looks wrong
  as an open point instead of editing it.
- **Pin the source.** The source of a page is a file at a specific commit (or
  blob) in a repository, not "the latest version". If the source is still on an
  unmerged branch, say so in the request and in the report.
- **Back up before every write.** Save the current CMS item before changing it
  and prepare the file that restores it. No write without a way back.
- **Use the revision token.** `emdash content update` requires `--rev` from a
  fresh `emdash content get`. It exists so you do not overwrite a change you
  have not seen; never work around it with an old token.
- **Credentials stay out of files and messages.** Log in with `emdash login`
  or pass a token from the environment (`-t` / `--token`); do not write tokens
  into JSON files, scripts, commit messages, requests or reports. Do not paste
  internal instance URLs or IDs into public places.
- **The public site is read-only for checks.** Verification uses GET requests
  and a browser only.
- **Content and code are separate.** Replacing the body of an existing page or
  post is a CMS change and needs no deploy. A new kind of URL, a redirect or a
  sitemap rule is a code change to the site and goes through the site's own
  deploy flow (see step B7).

## A. Requesting a change (requester side)

### A1. Compare the source with what is live

1. Fetch the current public page (`curl -s <page URL> -o live.html`) and keep
   the file.
2. Convert both the live page and the source HTML to text and diff them. List
   what will be added, changed and removed (sections, paragraphs, table rows).
3. Pick **check phrases** from the diff:
   - phrases that must **appear** after the change (new section titles, new
     service names);
   - phrases that must **disappear** (sentences the new version removes).

### A2. Write the request

Send one request per change, containing:

```
Pages:          <public URL of each page to replace or add>
Source:         <repository> @ <commit>  <path of each source file>
What changes:   <short list from the diff in A1>
Check after:    HTTP 200 on each URL
                appears:    "<phrase>", "<phrase>"
                disappears: "<phrase>"
Unmerged?:      <no / yes: branch and PR>
Notes:          <anything the operator must not change, e.g. draft notice>
```

Send it through the channel the site operator actually reads. If that channel
is unavailable (for example its authentication has expired), record that the
request was not sent and wait; do not route the change around the operator.

### A3. Follow up

1. Check the public pages at agreed times with the checks from A2 (step C1
   gives a small script). Record the time and the counts for each check.
2. Until the "disappears" phrases are gone and the "appears" phrases are
   present, the change is **not live**. Do not report it as done.
3. If nothing has changed and you cannot see that the request arrived, re-send
   it **once**, saying it is a re-send and may be ignored if it is a duplicate.
   Do not keep re-sending; after that, report the state and wait for
   instructions.
4. When the checks pass, report completion to whoever asked for the change.

## B. Publishing with the emdash CLI (operator side)

Run the CLI from the site project (it is in `node_modules/.bin/emdash`) and
point it at the instance with `-u <instance URL>` or the configured default.
Keep everything below in one work folder per change.

### B1. Take in the source

1. Copy the source files from the pinned commit into the work folder, with a
   note of the repository, commit and file paths (and blob IDs if you have
   them).
2. Convert the HTML for the CMS: to Markdown (the CLI converts Markdown in
   rich-text fields to Portable Text when writing), or directly to Portable
   Text when you need blocks Markdown cannot express, such as images with
   captions. **Keep every character of the text.** The
   site's own layout supplies the look; do not carry over the source's CSS.
   Keep the conversion script with the change so the next replacement uses the
   same one.

### B2. Look at the current item

```bash
emdash content list pages --json                    # find the item and its slug
emdash content get pages <slug-or-id> --json        # current data and _rev
emdash content get pages <slug-or-id> --raw --json > before.raw.json
```

- `--raw` returns the body as stored (raw Portable Text) instead of Markdown.
- `--published` returns only the published data, ignoring a pending draft.
- From `before.raw.json`, write `restore.json` (the fields you will change,
  with their current values). That is the rollback.
- For a new item, look at an existing item of the same kind first and copy its
  conventions: slug pattern, title pattern, heading levels, how links open,
  the date field the listing sorts by, excerpt.

### B3. Write

**New item** — create as a draft, check, then publish:

```bash
emdash content create pages --slug <slug> --file new.json --draft --json
emdash content get pages <slug> --raw --json > draft.raw.json   # compare with new.json
emdash content publish pages <slug>
```

**Existing item** — update with the revision token. Either stage it as a
draft and publish after checking, or update directly (without `--draft` the
update is published immediately):

```bash
emdash content update pages <slug> --rev <_rev from get> --file new.json --draft
emdash content get pages <slug> --raw --json > draft.raw.json   # compare with new.json
emdash content publish pages <slug>
```

- Compare the draft with what you put in, field by field (title, body, slug,
  excerpt, dates). Publish only when they match.
- If a write fails with a network error, run `emdash content get` again and
  check whether `_rev` changed before retrying; do not assume it did or did
  not apply.
- `--override-lock` writes over another editor's edit lock. Do not use it
  unless the person holding the lock agrees.
- After publishing, save the item again (`after.raw.json`) and note the live
  revision and version. There should be no pending draft left.

### B4. Posts

Posts work the same way with the posts collection (`emdash content ... posts`).
In addition:

- Follow the structure of existing posts (lead sentence, heading level for
  sections, list style, link behaviour) rather than inventing a new one.
- Set the field the post listing sorts by, so the post appears where intended.
- After publishing, check the listing page, the RSS feed and the sitemap as
  well as the post itself (step C2).

### B5. Images

1. Use only images supplied for the purpose. Look at each one before upload
   (no personal data, no debug overlays, current UI).
2. Size them for the display width: about twice the width they are shown at,
   aspect ratio unchanged, no cropping. JPEG for photographic images, PNG for
   images that are mostly text. Keep the originals in the work folder.
3. Upload with alt text and caption:

   ```bash
   emdash media upload <file> --alt "<alt text>" --caption "<caption>" --json
   ```

4. Add image blocks to the body at the agreed places and update the item as in
   B3. Check that the existing blocks are unchanged (same keys, same text) and
   that only the image blocks were added.

### B6. Rollback

Write the rollback commands into the record before you finish:

```bash
emdash content update pages <slug> --rev <latest _rev> --file restore.json   # previous text
emdash content unpublish pages <slug>                                         # take a page down
emdash media delete <media-id>                                                # remove an uploaded image
```

### B7. When the site code must change

Some requests need more than content: a new URL pattern (for example a
`/terms/<name>/` route next to an existing `/privacy/<name>/`), a redirect from
the CMS's flat URL, or a sitemap rule. Then:

1. Confirm that your local site source is what is currently deployed before
   changing it, so you do not ship someone else's unfinished work.
2. Keep copies of the files you change and their diffs.
3. Deploy to a preview first and run the checks in C against the preview.
   Promote to production only after they pass, and record the previous
   deployment version for rollback. (On Cloudflare Workers, for example:
   upload a version, check its preview URL, then deploy that version.)
4. Prefer a rule that covers the next page of the same kind (CMS slug prefix →
   URL pattern) over a one-off route, so later pages need only a CMS change.

## C. Verifying on the public site

### C1. Status and check phrases

```bash
for u in <page URL> ...; do
  curl -s "$u" -o page.html -w "$u %{http_code}\n"
  for w in "<appears phrase>" "<disappears phrase>"; do
    printf '  %s=%s\n' "$w" "$(grep -o "$w" page.html | wc -l)"
  done
done
```

- Each page returns 200, with and without the trailing slash.
- Redirects that should exist still return 301 (for example the CMS's flat URL
  to the public URL, apex to www).
- "Appears" phrases count ≥ 1, "disappears" phrases count 0.

### C2. Page details

- `canonical` points to the page's own public URL; `noindex` is present only
  if it was asked for.
- The page is in the sitemap (and, for posts, in the post listing and RSS).
- Anchors that other systems link to (for example a data-deletion section
  linked from an app store listing) exist and land at the top of the section.

### C3. Text matches the source

Extract the headings, paragraphs and list items inside `<main>` from the live
page and from the source HTML, in order, as (tag, list depth, text), and
compare them. Report the number of items and whether they match exactly. A
spot check of a few sentences is not enough for a legal page.

### C4. Nothing else changed

Save the other pages of the same kind (other policies and terms, top page,
contact page) before the change and compare after. They should be unchanged
byte for byte, or the difference should be explained.

### C5. Layout

Open the page in a browser at a phone width (360px) and a desktop width
(1280px):

- no horizontal overflow (`document.documentElement.scrollWidth` equals the
  viewport width);
- tables, long URLs and images stay inside the content area;
- images load (`naturalWidth` > 0) — scroll to them before taking a
  screenshot, since lazy-loaded images can look blank in a full-page capture;
- take screenshots and look at them.

## Report

Use this shape:

```
Change:      <add / replace / post / images>  requested by <who>
Pages:       <public URLs>  status=<200 ...>  redirects=<ok/...>
Source:      <repository> @ <commit>  <paths>   (unmerged: <no/yes>)
CMS:         <collection>/<slug>  revision <before> → <after>  draft left=<no>
Code:        <none / what changed, preview checked, deployed version, previous version>
Verified:    text match=<n items, exact/diff>  check phrases=<ok/...>
             other pages unchanged=<yes/...>  360/1280 overflow=<none/...>
Records:     <work folder: source copy, before/new/draft/after/restore JSON, checks>
Rollback:    <commands from B6 / B7>
Open points: <things noticed but not changed, decisions needed>
```

On the requester side, report each follow-up check (time, status, counts) and
whether the request was sent or re-sent, until the change is live.
