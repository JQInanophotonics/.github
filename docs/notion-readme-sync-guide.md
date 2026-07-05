# Notion README-sync setup guide

How to wire up a GitHub Action that mirrors a repo's `README.md` into a
Notion page as real blocks (headings, tables, lists, code) rather than a
link or a stale copy-paste. Built and proven out in `QuickStartGit`
first — read that repo's `.github/workflows/notion-sync.yml` and
`.github/scripts/sync-readme.js` directly if anything here is unclear;
this doc summarizes them but they're the source of truth.

This assumes the repo already has the
[README banner treatment](readme-banner-style-guide.md) — the sync
script specifically knows how to convert that treatment's SVG banners
into Notion headings. If a repo's README is still plain `##` headings
(no banners), most of this still works, you just don't need the
banner-specific regex steps below.

## Prerequisites (one-time, per Notion page)

1. Create (or reuse) a Notion internal integration at
   [notion.so/my-integrations](https://www.notion.so/my-integrations),
   copy its secret token.
2. Create the target Notion page, then **share it with the integration**
   (`•••` menu → Connections → add the integration). Without this share
   step the API calls 404/403 regardless of a correct token.
3. Copy the page ID out of its URL (the 32-char hex string, with or
   without dashes — Notion accepts either).
4. In the GitHub repo: Settings → Secrets and variables → Actions, add:
   - `NOTION_TOKEN` — the integration secret from step 1
   - `NOTION_PAGE_ID` — the page ID from step 3

## Files to add

### `.github/workflows/notion-sync.yml`

```yaml
name: Sync README to Notion
on:
  workflow_dispatch:
  push:
    branches: [main]
    paths: ['README.md']

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install @notionhq/client @tryfabric/martian
      - run: node .github/scripts/sync-readme.js
        env:
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
          NOTION_PAGE_ID: ${{ secrets.NOTION_PAGE_ID }}
```

**The `paths: ['README.md']` filter means this only fires on commits that
touch `README.md`.** A commit that changes only the sync script itself
(`sync-readme.js`) will *not* auto-trigger a sync — use "Run workflow"
(`workflow_dispatch`) manually after editing the script, or bump
`README.md` (even a trivial whitespace change) to force it.

### `.github/scripts/sync-readme.js`

Start from `QuickStartGit`'s version and change exactly two things per
repo: the `rawBase`/`blobBase` URLs (repo name in the path). Everything
else is generic:

```js
const fs = require('fs');
const { Client } = require('@notionhq/client');
const { markdownToBlocks } = require('@tryfabric/martian');

const notion = new Client({ auth: process.env.NOTION_TOKEN });
const pageId = process.env.NOTION_PAGE_ID;

const rawBase  = 'https://raw.githubusercontent.com/JQInanophotonics/<REPO>/main/';
const blobBase = 'https://github.com/JQInanophotonics/<REPO>/blob/main/';

let md = fs.readFileSync('README.md', 'utf8');

// 1. The top header banner (assets/header.svg) just repeats the repo title -
//    Notion's own page title already covers that, so drop it entirely
//    rather than sync it as a heading or an image.
md = md.replace(
  /<picture>(?:(?!<\/picture>)[\s\S])*?<img\s+src="assets\/header\.svg"[^>]*\/>\s*<\/picture>\n?/g,
  ''
);

// 2. SVG section banners -> H2. Each banner is a <picture> whose <img> src
//    points at a local assets/banner-*.svg file, with alt text set to the
//    exact heading text (README.md keeps alt clean of the "NN — " index
//    prefix shown inside the SVG itself, so no stripping is needed here).
md = md.replace(
  /<picture>(?:(?!<\/picture>)[\s\S])*?<img\s+src="assets\/banner-[^"]*"[^>]*\balt="([^"]*)"[^>]*\/>\s*<\/picture>/g,
  '## $1'
);

// 3. Badge links (small <picture> images pointing at img.shields.io, wrapped
//    in an <a>) carry no content worth syncing to Notion - drop them.
md = md.replace(
  /<a\s+href="[^"]*">(?:(?!<\/a>)[\s\S])*?img\.shields\.io(?:(?!<\/a>)[\s\S])*?<\/a>\n?/g,
  ''
);

// 4. Centering wrapper <div>s around the header banner/badge row - just
//    layout, meaningless once steps 1-3 have run.
md = md.replace(/<div align="center">\s*\n?/g, '');
md = md.replace(/<\/div>\s*\n?/g, '');

// 5. In-page anchors (e.g. <a id="pages"></a>, used for GitHub badge links) -
//    Notion has no use for them.
md = md.replace(/<a\s+id="[^"]*">\s*<\/a>\n?/g, '');

// 6. Images: relative srcs -> raw.githubusercontent.com (actual file bytes,
//    so Notion can render them as image blocks). Any real markdown images
//    left after step 2 (i.e. not banners) get this treatment.
md = md.replace(/(!\[[^\]]*\]\()(?!https?:\/\/)(?:\.\/)?([^)]+)\)/g, `$1${rawBase}$2)`);

// 7. Regular links: relative -> absolute GitHub blob URLs.
//    (?<!!) ensures image syntax from step 6 is not re-touched.
md = md.replace(/(?<!!)(\[[^\]]*\]\()(?!https?:\/\/|#)(?:\.\/)?([^)]+)\)/g, `$1${blobBase}$2)`);

const blocks = markdownToBlocks(md);

(async () => {
  // Clear existing page content
  let cursor;
  const existing = [];
  do {
    const res = await notion.blocks.children.list({ block_id: pageId, start_cursor: cursor });
    existing.push(...res.results);
    cursor = res.has_more ? res.next_cursor : undefined;
  } while (cursor);
  for (const b of existing) {
    await notion.blocks.delete({ block_id: b.id });
  }

  // Append in chunks (API limit: 100 blocks per request)
  for (let i = 0; i < blocks.length; i += 100) {
    await notion.blocks.children.append({
      block_id: pageId,
      children: blocks.slice(i, i + 100),
    });
  }
  console.log(`Synced ${blocks.length} blocks.`);
})();
```

## Companion requirement in `README.md` itself

Each section banner's `<img alt="...">` must be the **clean heading
text only** — no "NN — " index prefix, even though the SVG image itself
still shows the number visually. The script uses alt text verbatim as
the Notion heading; if the prefix is left in, Notion headings will read
"00 — Forewords" instead of "Forewords". Example of what README.md
should have:

```html
<picture><source media="(prefers-color-scheme: dark)" srcset="assets/dark/banner-forewords.svg"/><img src="assets/banner-forewords.svg" width="97%" alt="Forewords"/></picture>
```

not `alt="00 — Forewords"`.

## Verifying before you rely on it

Don't trust a Notion sync blind on the first run — dry-run the transform
locally first (no Notion credentials needed, it's pure string
processing):

```bash
node -e "
const fs = require('fs');
let md = fs.readFileSync('README.md', 'utf8');
// ...paste the md.replace(...) steps from sync-readme.js here...
fs.writeFileSync('README.transformed.md', md);
"
grep -n '^##\|<picture>\|<div\|<a ' README.transformed.md
```

Confirm: every section has exactly one `##` heading, no `<picture>`,
`<div`, or `<a ` tags remain, and the header banner produced no heading
at all. Then (optional but cheap) install `@tryfabric/martian` locally
and run `markdownToBlocks()` on the transformed file to confirm it
produces the expected number of `heading_2` blocks before pushing.

## Known gotchas

- **Page must be shared with the integration**, not just created by the
  same Notion account — this is the #1 cause of a 403 on first run.
- **`workflow_dispatch` doesn't auto-run on script-only commits** — see
  the `paths:` filter note above.
- **Full-page replace, not a diff/merge.** The script deletes every
  existing block on the page and re-appends from scratch each run — do
  not expect manual edits made directly in Notion to survive the next
  sync. If a page needs space for human-added content Notion-side that
  shouldn't be clobbered, put that content on a *child* page linked from
  the synced page, not on the synced page itself.
- **Badge/banner detection is regex-based on the literal HTML markup**
  the banner style guide produces. If a repo's README uses a
  differently-shaped banner (different attribute order, self-closing
  vs. not, etc.), the regexes may need adjusting — verify with the dry
  run above rather than assuming it "just works" on a new repo.
