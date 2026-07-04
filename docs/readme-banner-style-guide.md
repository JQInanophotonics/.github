# README banner style guide

This is a self-contained prompt for a fresh agent (no prior context) to add
light/dark SVG section banners to another repo's `README.md`, matching the
style built for `ScientificPresentations`. Paste the section below
("Prompt starts here") into a new agent/session working in the target repo.

The design was arrived at through several rounds of real user feedback —
those corrections are baked into the rules below as hard requirements, not
suggestions. Skipping any of them reproduces a mistake that already
happened once.

---

## Prompt starts here

You are giving this repo's `README.md` a banner treatment: a light/dark SVG
image above each top-level (`##`) section, in the JQInanophotonics org's
established visual identity. This has already been done in the
`ScientificPresentations` repo — that repo's `assets/*.svg` files and
`README.md` are the reference implementation; match them exactly in style,
adapting only the content (title, section names, badges) to this repo.

### Visual identity (do not deviate — these are the org's real brand values)

Colors (from `beamerthemeJQInanophotonics.sty` in `JqiNanoBeamerTemplate`):

| Token | Hex | Use |
|---|---|---|
| UMD red (accent) | `#E21833` | the thin rule under the label, and under the header title |
| Near-black (light text) | `#1a1a1a` | label/title text in light mode |
| Off-white (dark text) | `#F5F5F5` | label/title text in dark mode |
| Ghost gray (light) | `#D5D5D5` | the large section numeral in light mode |
| Ghost gray (dark) | `#262b32` | the large section numeral in dark mode |
| Light background | `#FFFFFF` | light-mode banner background |
| Dark background | `#0d1117` | dark-mode banner background — this is GitHub's own dark-mode page background, chosen deliberately so the banner blends into the page instead of showing as a boxed rectangle |

Fonts (already in use in this org's talk repos — reference via CSS font
stacks, do not embed font files):

- Label/title text: `'Helvetica Neue Condensed','Helvetica Neue',Arial,sans-serif`
- Section numeral: `ui-monospace,'Monaspace Neon','SF Mono',Menlo,Consolas,monospace`

### Exact SVG structure

**Section banner** (one per `##` heading), `viewBox="0 0 1200 116"`:

1. Background `<rect>` filling the canvas, light or dark bg color.
2. The section number (`"00"`, `"01"`, ...) at `x=40`, mono font,
   `font-size="54"`, `font-weight="600"`, ghost-gray fill,
   `dominant-baseline="central"` at `y = H/2`.
3. The section title, uppercased, immediately after the numeral (`x` =
   numeral's x + measured numeral width + 26), headline font,
   `font-size="37"`, `font-weight="700"`, `letter-spacing="2"`, main text
   color, same `dominant-baseline="central"` / `y = H/2`.
4. A thin horizontal `<line>` in UMD red, `stroke-width="2"`, from
   (label's x + measured label width + 32) to `W - 44`, at `y = H/2`.
5. **Nothing else.** No colored block/box behind the numeral. No text on
   the right-hand side (no breadcrumb, no file path, no anchor). A previous
   attempt added a right-aligned `README.md#anchor`-style breadcrumb and a
   solid-color box behind the numeral — both were explicitly rejected by
   the user as clutter. Do not reintroduce either.

**Header banner** (one, for the repo title), `viewBox="0 0 1200 120"`:

1. Background `<rect>`.
2. The repo's display title, centered, headline font, `font-size="50"`,
   `font-weight="700"`, `letter-spacing="3"`, `text-anchor="middle"`,
   `dominant-baseline="central"`, at `y = H/2`.
3. A short centered rule in UMD red under the title (`stroke-width="1.5"`,
   centered, ~320px long).
4. **No subtitle/tagline line underneath.** An earlier version added a
   descriptive tagline here; the user flagged it as redundant with the
   prose immediately below it in the README. Leave the header to the title
   and rule only, unless there is something the prose genuinely does not
   already say.

Generate both a light file (`assets/`) and a dark file (`assets/dark/`) per
banner — two full SVG files, not one file with an embedded
`@media (prefers-color-scheme)` block (GitHub's `<picture>`/`<source>`
switching, below, handles that at the markup level instead).

### Critical: measure text width, never estimate it

The line's start x-coordinate depends on the label's rendered width. Do not
guess a per-character width. Use a real font-metrics library
(`PIL.ImageFont.getlength`, or equivalent) against a **plain, non-condensed
bold sans font** (e.g. `/System/Library/Fonts/Supplemental/Arial Bold.ttf`
on macOS) — deliberately wider than the actual condensed brand font, so the
computed line position always has *more* clearance than needed, never less.
An earlier version estimated width with a fixed multiplier and the line
rendered through the middle of longer labels. Write a small script (Python
+ PIL is sufficient) that computes this per label rather than hand-picking
coordinates.

### Critical: verify at the real render width, not the SVG's own coordinate space

The SVG's `viewBox` is 1200 units wide, but it is placed at `width="97%"`
of GitHub's markdown body column, which is roughly 800px wide in practice —
so everything in the banner renders at roughly **0.7×** the size the
viewBox numbers suggest. A banner that looks correctly proportioned when
you preview the raw 1200px-wide SVG will look undersized and close to
body-text size once actually on the page. Before treating any banner as
final:

1. Render it with `rsvg-convert -w 800 banner.svg -o preview.png` (matching
   the real display width, not the viewBox width).
2. Render a comparison line of plain 16px text (any tool — a simple PIL
   canvas is enough) at the same pixel width.
3. Stack them and look at the image. The banner's label text must read
   clearly larger than the 16px line — not merely legible, obviously a
   heading. If it's close in size to the body-text line, the font sizes in
   the SVG are too small; scale them up and re-check.

Do this check before wiring banners into the README, and again after any
change to font sizes. Skip it if you're reusing font sizes already
validated in the reference implementation unchanged — the check exists to
catch scale regressions, not to re-prove settled values on every repo.

### Wiring into `README.md`

For each existing `## Section Title` heading:

```markdown
<picture><source media="(prefers-color-scheme: dark)" srcset="assets/dark/banner-SLUG.svg"/><img src="assets/banner-SLUG.svg" width="97%" alt="NN — Section Title"/></picture>

(the section's existing prose, immediately after — no blank heading line)
```

**The banner replaces the `##` heading — it does not sit above or below a
second, separately-visible plain-text heading.** Delete the `## Section
Title` line entirely once its banner is in place. Do not leave both; a
banner immediately followed by a literal repeated heading was flagged
directly as "fucking redundant" in the repo this pattern originated from.
`SLUG` is a short kebab-case name for the section (`forewords`,
`structuring-a-talk`, etc.) — reuse it consistently for both the light and
dark asset filenames.

**If a top badge links to `#SLUG` (e.g. a "PAGES" badge linking to a Pages
section), add an explicit `<a id="SLUG"></a>` immediately above that
section's banner.** Deleting the literal `## Section Title` heading also
deletes the only thing that gives GitHub's auto-generated `#slug` anchor
to exist — without the explicit anchor, the badge silently 404s. This bit
the original `ScientificPresentations` implementation (its `#pages` badge
was broken until this was caught while applying the pattern to
`ScientificGraphicDesign`) — check for this on every repo this pattern is
applied to, not just the one where it was first found.

At the very top of the README, before the first banner:

```markdown
<div align="center">

<picture><source media="(prefers-color-scheme: dark)" srcset="assets/dark/header.svg"/><img src="assets/header.svg" width="97%" alt="Repo Title"/></picture>

<a href="LINK1"><picture><source media="(prefers-color-scheme: dark)" srcset="https://img.shields.io/badge/LABEL-0d1117?style=flat-square&logoColor=ffffff"/><img src="https://img.shields.io/badge/LABEL-ffffff?style=flat-square&logoColor=1a1a1a" alt="Label"/></picture></a>
(one such badge per relevant link — sibling repos, prerequisite repos, key pages — 3-5 is about right)

</div>
```

Badge background matches the page background per theme (`0d1117` dark /
`ffffff` light) so badges read as flat labels rather than boxed buttons,
exactly like the header/section banners blending into the page. Pick
badge targets relevant to *this* repo (its own sibling repos, its own
prerequisite repos, an in-page anchor like `#pages` if this repo has an
index table) — do not copy `ScientificPresentations`'s badge list
verbatim if the links don't apply here.

### Before finishing

- Run a link/asset checker confirming every `src=`/`href=` in the modified
  README resolves to a real file or a real URL (relative asset paths are
  the easy thing to typo).
- Confirm there are zero remaining `## ` headings for sections that now
  have a banner (`grep -c '^## ' README.md` after the edit — every
  banner'd section should have removed its heading, so this count should
  only include sections that deliberately have no banner, if any).
- Do the width/scale verification (above) at least once on the final
  assets before committing.
- Commit with a message that says what repo-specific content changed
  (title, sections, badge targets) — not a generic "add banners" message.

---

## Prompt ends here

## Why this file exists (rule, not just a one-off prompt)

Treat the checklist above as the standing style rule for any
JQInanophotonics repo's `README.md` banner treatment, not a one-time
instruction. If this pattern is applied to a new repo and a genuine
improvement is found, update **this file** (and re-sync
`ScientificPresentations`'s own banners if the change is generally
better) rather than letting each repo's banners drift into a different
style. The canonical reference implementation remains
`ScientificPresentations/assets/*.svg` and `ScientificPresentations/README.md`
— when in doubt about an exact value, read those files directly rather
than relying on this document's summary of them.
