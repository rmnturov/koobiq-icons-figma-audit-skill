---
name: koobiq-icons-figma-audit
description: >-
  Validate that a Figma icon component is organized correctly before exporting it from the Koobiq
  icons file (figma:sync / figma-export). Checks node type, name format, size, layout, the root
  shape/shape-2 structure, scale constraints, fill color contract, and the description metadata —
  and verifies size variants of the same icon agree with each other. Use this whenever the user
  wants to check, review, validate, or "проверить" a newly drawn or unpublished icon component
  against the library conventions, or asks whether an icon is ready to export — even if they only
  paste a Figma link or an icon name. Run it before figma:sync when adding or editing icons.
---

# Koobiq Icons Figma Audit

This skill verifies that an icon **component** in the Koobiq icons Figma file follows the library's
structural conventions, so that `figma:sync` (the `@figma-export` pipeline) produces a clean,
correct SVG. Catching problems in Figma is far cheaper than discovering them after export, because
the export silently transforms whatever it finds — a wrong layer name, an extra node, or an
off-grid coordinate becomes a broken SVG with no error.

It is a **read-only audit by default**: inspect, then report a clear pass/fail checklist with
specific, actionable findings. Only modify the Figma file if the user explicitly asks you to fix
something (see "Offering fixes").

## Prerequisites

- The Figma MCP must be connected. This skill drives it through `get_metadata` (structure overview),
  `use_figma` (precise inspection via the Plugin API), and `get_screenshot` (visual sanity check).
- **Before any `use_figma` call, load the `figma-use` skill** — it documents the Plugin API rules and
  prevents common, hard-to-debug failures.
- The file key and size pages are fixed (verified against the live document). To skip a page-discovery
  round-trip, jump straight to a page by its node id:

  | size | page node id |
  | ---- | ------------ |
  | 16   | `1320:17387` |
  | 24   | `42149:860`  |
  | 32   | `42149:862`  |
  | 48   | `42149:861`  |
  | 64   | `42149:863`  |

  File key: `WvUarBj8tO256vML4PkBLl`. Each page name is the size suffix; an icon named `foo_24` lives on
  page `24`. Document:
  https://www.figma.com/design/WvUarBj8tO256vML4PkBLl/%F0%9F%9F%A5-koobiq-%C2%B7-icons?node-id=1320-17387
  - This file key is the source of truth — it matches `config.mjs`. The "Draw an icon" link in
    `packages/icons/README.md` points at a **different, wrong file** — ignore it and trust the key above.
  - `get_metadata` without a `nodeId` lists only the first page (`ICONS`, which holds no icon
    components) — it does **not** enumerate the size pages. Don't use it for page discovery; use the
    node ids above.

### If the file can't be accessed

If a Figma call fails — auth/permission error, "node does not exist", an empty/unexpected file, or the
MCP not responding — **stop and surface it to the user instead of guessing**. Tell them what failed and
ask them to do one of:
- provide a different / correct Figma link (the file key in `config.mjs` may be stale, or they pasted a
  link to another file or a branch), or
- re-check the `FIGMA_TOKEN` in the repo `.env` (the `figma-export` / MCP access token may be missing,
  expired, or revoked), or
- confirm their Figma account has access to the file (it's a private design file — they may need to be
  invited or use a token tied to an account with permission).

Don't fabricate an audit result from assumptions; a check is only meaningful against the live file.

## Parallel reads and page context

Figma read calls may run in parallel, but only **before the first Figma edit** of the task. After any
edit has started, run Figma calls one at a time.

Parallel reads that switch pages can interfere with each other. In one observed run, a read that
targeted page 24 returned page 16's data (the same component count, wrong names), and the audit
reported the 24px variants as missing. Guard every read that switches pages:

```js
// Use in every read that switches pages. Returns data only for nodes that belong to PAGE_ID.
const PAGE_ID = "42149:860";   // page node id from the table above
const NAMES = ["foo_24"];       // component names to look for
const page = await figma.getNodeByIdAsync(PAGE_ID);
await figma.setCurrentPageAsync(page);
if (figma.currentPage.id !== PAGE_ID) throw new Error("page context mismatch before read");
const pageOf = (n) => { let p = n; while (p && p.type !== "PAGE") p = p.parent; return p; };
const hits = figma.currentPage
  .findAllWithCriteria({ types: ["COMPONENT", "COMPONENT_SET"] })
  .filter((n) => NAMES.includes(n.name) && pageOf(n) && pageOf(n).id === PAGE_ID)
  .map((n) => ({ name: n.name, id: n.id, w: n.width, h: n.height }));
if (figma.currentPage.id !== PAGE_ID) throw new Error("page context changed during read");
return { pageId: PAGE_ID, hits };
```

If a guard throws, discard that result and re-run the read. A "missing" claim (for example, "no 24px
variant") must rest on a guarded read of that page, or on a lookup by node id. Never report absence
from a read that threw or from a page you did not check.

## Step 1 — Resolve the target

The user may give you any of:

- **A Figma node URL or id** (e.g. `node-id=44731-6643`) → inspect exactly that node, then also look up
  its sibling size variants by name for the cross-variant check.
- **An icon name** (e.g. `floating-window`, or `floating-window_16`) → find every size variant across
  the icon pages and check them all. New icons are often **unpublished and may even be unnamed**, so if
  a name match fails, list the components on the relevant page(s) and ask the user to point at the node,
  or inspect by node id.

Page names equal the size suffix. **Size sweep:** check all five pages (16 / 24 / 32 / 48 / 64) for
every requested name, because an icon may exist at sizes the user did not mention. Use the page node
ids from the Prerequisites table, one `use_figma` call per page, each with the guard from "Parallel
reads and page context". Run them in parallel only before any Figma edit. Report the result as a table
(`name × size → node id or missing`).

## Step 2 — Inspect

Run this read-only script (adapt the `targets`/page id). It returns everything the rules below need.
Pair it with a `get_screenshot` so you can confirm the glyph actually renders.

```js
// Paste into use_figma (skillNames: "figma-use"). Set PAGE_ID and NAMES.
// Page node ids: 16=1320:17387  24=42149:860  32=42149:862  48=42149:861  64=42149:863
const PAGE_ID = "1320:17387";      // page whose name is the size, e.g. "16"
const NAMES = ["floating-window_16"]; // component name(s) to audit on this page
figma.skipInvisibleInstanceChildren = false; // false: rule G must see hidden layers too
const page = await figma.getNodeByIdAsync(PAGE_ID);
await figma.setCurrentPageAsync(page);
if (figma.currentPage.id !== PAGE_ID) throw new Error("page context mismatch before read");

const toHex = (c) => {
  const h = (n) => Math.round(n * 255).toString(16).padStart(2, "0").toUpperCase();
  return "#" + h(c.r) + h(c.g) + h(c.b);
};
const describeFill = (n) => {
  if (!n.fills || n.fills === figma.mixed) return n.fills === figma.mixed ? "MIXED" : null;
  return n.fills.filter((f) => f.visible !== false).map((f) => ({
    type: f.type,
    hex: f.type === "SOLID" ? toHex(f.color) : undefined,
    opacity: f.opacity,
    boundVar: f.boundVariables && f.boundVariables.color ? f.boundVariables.color.id : null,
  }));
};

const pageOf = (n) => { let p = n; while (p && p.type !== "PAGE") p = p.parent; return p; };
const all = page.findAllWithCriteria({ types: ["COMPONENT", "COMPONENT_SET"] })
  .filter((n) => pageOf(n) && pageOf(n).id === PAGE_ID);
const report = [];
for (const name of NAMES) {
  const c = all.find((x) => x.name === name);
  if (!c) { report.push({ name, found: false }); continue; }
  report.push({
    name: c.name, id: c.id, type: c.type,
    width: c.width, height: c.height, x: c.x, y: c.y,
    layoutMode: c.layoutMode,
    clipsContent: c.clipsContent,
    description: c.description,
    children: c.children.map((ch) => ({
      name: ch.name, type: ch.type,
      w: ch.width, h: ch.height, x: ch.x, y: ch.y,
      constraints: ch.constraints,
      fills: describeFill(ch),
      styles: { fill: ch.fillStyleId, stroke: ch.strokeStyleId, effect: ch.effectStyleId },
      boundVariableFields: ch.boundVariables ? Object.keys(ch.boundVariables) : [],
    })),
  });
}
if (figma.currentPage.id !== PAGE_ID) throw new Error("page context changed during read");
return report;
```

To validate the **color contract** reliably, also read a known-good sibling icon of the same size
(e.g. `list_16`) the same way and compare the primary shape's bound variable **key** (Rule D).
The nominal contract hex and the value actually stored in the file differ (see Rule D), so comparing
against a real neighbor is more trustworthy than matching a literal hex.

## Step 3 — Check against the rules

Report each as ✅ pass / ❌ fail with the observed value and, on failure, what it should be.

### A. Node type & name
- The node is a **`COMPONENT`** (or a `COMPONENT_SET` of size variants), not a `FRAME`, `GROUP`, or
  `INSTANCE`. Only components export.
- **First character — check this first, it changes what the name means:**
  - Starts with **`.` or `_`** → an **internal component, intentionally NOT exported.** The export's
    `filterComponent` in `.figmaexportrc.mjs` drops any name matching `^[._]`. This is by design, not a
    bug — don't audit it as a broken icon. Only raise it if the user expected this node to export (then
    the fix is to rename it without the leading `.`/`_`).
  - Starts with **`-`** → **invalid, not allowed.** Flag as a failure.
  - Starts with a **digit** → allowed but **discouraged.** It still exports (e.g. `3-columns_32`), but a
    name should preferably begin with an English letter — suggest the user rename it. Soft
    recommendation, not a blocker.
  - Starts with an **English lowercase letter** → the preferred case.
- **Name format** — an exportable icon name must match this regex:

  ```
  ^[a-z0-9]+(-[a-z0-9]+)*_(16|24|32|48|64)$
  ```

  English lowercase letters and digits only; words joined by **single** hyphens (no `--`, no
  leading/trailing hyphen); then **exactly one** underscore; then a valid size suffix. No uppercase,
  spaces, Cyrillic/other scripts, or other punctuation.
  - ✅ `floating-window_16`, `api_24`, `arrow-up-to-bracket_16` (and, format-valid but digit-discouraged,
    `3-columns_32`)
  - ❌ `Floating-Window_16` (uppercase) · `плавающее-окно_16` (not English) · `floating_window_16` /
    `icon__16` (two underscores) · `floating--window_16` (double hyphen) · `floating-window-16` (hyphen
    instead of underscore before size) · `-floating-window_16` (leading hyphen) · `floating-window_20`
    (size not in the allowed set) · `floating-window_16 ` (trailing space)
- **`-o` suffix = outline variant.** The name without `-o` is the filled variant. A pair such as
  `cloud_24` / `cloud-o_24` or `cloud-arrow-up_24` / `cloud-arrow-up-o_24` is one icon in two styles.
  That is valid and is not a conflict or a duplicate; don't ask for a rename because of it.
- The numeric suffix must equal the component's actual size (Rule B). A `..._16` that is 24×24 is a bug.

### B. Geometry & layout
- **No auto-layout**: `layoutMode === "NONE"`. Auto-layout reflows children and corrupts the export.
- **Fixed & square**: `width === height === size`. The size must be a whole number of pixels
  (pixel-grid aligned); fractional component sizes mean the icon was not drawn on the grid.
- **Clip content OFF**: `clipsContent === false`. The "Clip content" toggle must be disabled. It must be the same (off) for every size variant of the icon.
- **No transforms**: no rotation, flip, or scale on the component or any descendant. Check each node's
  `relativeTransform` `[[a, c, tx], [b, d, ty]]`: `b` or `c` ≠ 0 means rotation; `a*d - b*c < 0` means
  a flip (a mirror, e.g. `scaleY = -1`); `hypot(a, b)` or `hypot(c, d)` ≠ 1 means scale. Also check `rotation`.
  The icon must look the same without any transform.
  When it is on, any artwork extending past the frame bounds is silently cropped in the export,
  producing a clipped or truncated icon.

### C. Root structure (this is where most real bugs hide)
- **Single-color icon**: exactly **one** root child — a `VECTOR` named **`shape`**.
- **Two-color icon**: exactly **two** root children — **`shape`** (primary) and **`shape-2`**
  (secondary). Nothing else.
- **No other nodes** at the component root: no stray frames, groups, rectangles, slices, duplicate
  vectors, or hidden helper layers. Each extra node ends up in the exported SVG.
- Each shape's **constraints are `SCALE` on both axes** (`horizontal: "SCALE"`, `vertical: "SCALE"`)
  so the artwork scales with the frame.
  - **Severity: warning (hygiene), not an export blocker.** Constraints control how the component
    resizes in Figma, not the exported path data. Fix them with the other approved Figma edits, because
    instances and resizing depend on them.
- Ideally each shape is a single flattened path (see the repo README "Draw an icon" steps: outline,
  flatten, Non-zero fill rule). Flag multiple sub-paths or unflattened groups as a warning.

### D. Fill color contract
The contract (repo README "SVG Color Zones Guide"):
- **Primary zone `shape`** → black **`#21222C`**. The export removes its `fill` so the SVG inherits
  `currentColor`.
- **Secondary zone `shape-2`** → red **`#E21D03`**. The export rewrites it to `fill="currentColor"`.

Practical reality: in the production file the primary fill is bound to the shared **`Icon/Contrast`**
variable, which **resolves to `#000819` at 0.85 paint opacity** — *not* a literal `#21222C`. So:
- **Reference key: `91eb54e0eae961c540c994192c4eed18e4ea1825`** — `Icon/Contrast` from
  `🟥 koobiq · components 19` (collection key `c78fab428cf2003c9aa53d86b132a2fa95acb2e2`).
- Compare the primary paint's bound variable **key** with that reference
  (`(await figma.variables.getVariableByIdAsync(boundId)).key`), and check the paint opacity is 0.85.
  Don't compare names: `Icon/Contrast` exists in several libraries and collections. The **only**
  variable allowed in an icon is this one, by key. A same-named variable with a different key
  (from another library or collection) is a **failure**, not a warning. Fix: rebind to the reference
  key variable from the same library collection.
- Secondary `shape-2` must be either a solid literal `#E21D03` (the export matches it as a literal hex —
  `secondaryZoneColorHex` in `.figmaexportrc.mjs`), or bound by key to `Icon/Error` from
  `🟥 koobiq · components 19` (key `e736157dcefc2a3e016ec2ddd41a7dc8cf8a5bc6`). For the variable case,
  the paint opacity is not checked. Any other fill fails.
- A primary fill that is a plain literal color matching no sibling, or a secondary fill that is not
  `#E21D03`, will export with the wrong color behavior — flag it.

### G. Variables and styles (fail if present)
An icon component must not use variables or styles, except for one color binding:
- **Allowed:** exactly two variables, both by key from `components 19`:
  - primary `shape` fill: `Icon/Contrast`, key `91eb54e0eae961c540c994192c4eed18e4ea1825`;
  - secondary `shape-2` fill: `Icon/Error`, key `e736157dcefc2a3e016ec2ddd41a7dc8cf8a5bc6`.
  Nothing else, and no variable bound to strokes.
- **Forbidden on the component and every child, including nested layers:**
  - any other bound variable: `boundVariables` on the node (size, padding, radius, opacity, etc.), a
    paint bound to any variable other than the two allowed above (this includes strokes), or an
    effect bound to a variable;
  - any style: `fillStyleId`, `strokeStyleId`, `textStyleId`, `effectStyleId`, `gridStyleId` must be
    empty (`""`). A same-named style from another library counts as a style too.
- **Hidden paints:** no fill or stroke with `visible: false` on the component or on any child. Remove them.
- Report each violation with the node id, the field, and the variable or style name and key. Fix: unbind
  non-color variables (`node.setBoundVariable(field, null)`), detach styles (`node.fillStyleId = ""` and
  similar), and keep the bound color paint only on `shape`.

### E. Description metadata
The component `description` field uses this exact format — each block separated by a single blank line,
in this order:

```
kbq-{{name}}            ← line 1: REQUIRED  (kbq icon name)
                        ← REQUIRED blank line
{{tag1, tag2, tag3}}    ← tags, comma + space separated, alphabetical   (optional)
                        ← blank line
{{description}}         ← human-readable description       (optional)
                        ← blank line
pt-icons-{{name}}       ← name in the pt-icons package     (optional)
                        ← blank line
<11: {{old-name}}       ← pre-v11 name in @koobiq/icons     (optional)
```

**Required:**
- The `kbq-{{name}}` line — the icon's name in the `@koobiq/icons` package: the component name with a
  `kbq-` prefix, including the size suffix. E.g. component `floating-window_16` → `kbq-floating-window_16`.
  Flag a failure if it's missing, doesn't match the component name, or has the wrong suffix.
  The `kbq-` prefix is followed directly by the name: `kbq-name_16`. `kbq-icons-name_16` is wrong.
  If the description starts with a note instead of the `kbq-` line, insert the `kbq-` line above the note
  and keep the note; do not overwrite it.
- The blank line immediately after it.

**Optional** (absence is a soft note, never a blocker): the tags line, the description line, the
`pt-icons-{{name}}` line, and the `<11:` old-name line. When present, each is preceded by its own
blank-line separator, and tags must be comma-plus-space separated and **sorted alphabetically** (e.g.
`круг, крестик, список` — flag out-of-order tags and report the corrected order). Don't invent missing
content.

**The `<11:` old-name line** records the icon's name in `@koobiq/icons` **in versions below 11** — the
prefix reads literally as "< 11". The icon was renamed in the 11.0.0 release (a batch of breaking
renames), so this line preserves the name used in 10.x and earlier. When present it is the **last**
block in the description. Format: the literal prefix `<11: ` followed by the pre-v11 icon name
**including the size suffix**, with no `kbq-` / `pt-icons-` prefix — e.g. an icon now named
`circle-telegram_16` carries `<11: telegram-circle_16` (renamed in 11.0.0). It
is the source of truth for the matching entry in `mapping-interop.json`, which maps
`{current-name: old-name}` (e.g. `"circle-telegram_16": "telegram-circle_16"`). When auditing, if the
icon has a `mapping-interop.json` entry, the `<11:` value should equal that entry's value — flag a
mismatch. Most icons were never renamed and correctly have no `<11:` line; its absence is never a
blocker.

**Whitespace rules** (flag violations):
- Each separator is **exactly one** blank line — never two or more consecutive blank lines between blocks.
- The description has **no trailing whitespace**: it must not end with spaces, tabs, or trailing blank
  line(s), and individual lines should not have trailing spaces.

**Classifying a single middle block** (when there's only one content block between the `kbq-…` line and
the `pt-icons-…` line): if the line is **comma-plus-space separated** (`tag1, tag2, tag3`), treat it as
**tags**, not a description. A line with no `, ` separator is a description. Don't ask the user about
this case — apply the rule. Lean on Rule F (cross-variant agreement) as the stronger signal for whether
the content is correct.

### F. Cross-variant consistency (the part that actually matters for metadata)
When an icon has several size variants (`_16`, `_24`, `_32`, `_48`, `_64` — same base name, different
size suffix):
- They must agree on **structure** — all single-color or all two-color, same zone names.
- Their **description content (tags + description) must be identical** across every size. This is the
  important metadata check: it's fine for an icon to have no tags/description, but if it has them they
  must match across all sizes — only the size suffix in the `kbq-…` / `pt-icons-…` / `<11:` lines may
  differ. A `_24` whose tags/description disagree with its `_16` is a real bug worth flagging.
- If present, the **`<11:` old-name** must agree across variants too, differing only by the size suffix
  (`<11: telegram-circle_16` vs `<11: telegram-circle_24`) — a `_24` whose `<11:` base name disagrees
  with its `_16` is a bug.

> **Not yet checked in practice:** the cross-size consistency check (rule F, description and tags of the
> same icon across `_16`/`_24`/`_32`/`_48`/`_64`) has not been run on the whole file yet. A full run needs
> a description fingerprint per size page, compared across pages.

## Step 4 — Report

Lead with a one-line verdict (**ready to export** / **needs fixes**), then the per-rule checklist
grouped by variant, then a short "Fixes needed" list ordered by severity (structure/naming first,
metadata last). Include the node ids so the user can jump straight to a node. Mention which checks you
verified against a sibling icon.

## Offering fixes

If — and only if — the user asks you to fix the issues, you can do it with `use_figma` (load
`figma-use` first): rename the component, set `layoutMode = "NONE"`, set `clipsContent = false`,
set child `constraints`, rename
`shape`/`shape-2`, rebind the primary fill to the `Icon/Contrast` variable used by siblings, set the
secondary fill to `#E21D03` or to the `Icon/Error` variable, or rewrite the `description`. Re-run Step 2 afterward and screenshot to
confirm. Never edit the file as part of a plain "check" request. Show the exact change set (node id,
property, old → new) and apply only what the user approves.

### Binding the color variables

Use these keys (collection `c78fab42…` is `Color Styles` in `🟥 koobiq · components 19`):

| Role | Variable | Key |
| ---- | -------- | --- |
| primary `shape` | `Icon/Contrast` | `91eb54e0eae961c540c994192c4eed18e4ea1825` |
| secondary `shape-2` | `Icon/Error` | `e736157dcefc2a3e016ec2ddd41a7dc8cf8a5bc6` |

Bind by key: `const v = await figma.variables.importVariableByKeyAsync(KEY)`, then
`paint = figma.variables.setBoundVariableForPaint(paint, 'color', v)` and assign the new `fills` array.
Set the primary paint opacity to 0.85 if it differs. Do not bind any other variable.

### Removing transforms

1. **Before changing anything, check instances.** A child's rotation shows in every instance, even when the
   component's own rotation cancels it on the canvas. Resetting it can change the instance look.
2. **Rotation.** If the instance look does not change, set the component and child `rotation` to 0. If it
   changes, bake the rotation into the geometry first (step 3), then reset.
3. **Flip or rotation baked into geometry.** `figma.flatten([node], parent, index)` returns a new vector with
   the transform applied and identity matrix. Keep the name, parent, and index. Note that the node id changes:
   instance overrides on the old id can break, so check instances first.
4. **Verify.** For a flipped node, the visual top is `y − height` (`y` is the bottom edge after a vertical flip),
   so compare the visual box, not the raw `y`. Take `get_screenshot` of the component before and after, and
   compare the two PNGs pixel by pixel. Re-run the transform check; it must report no flagged nodes.

## Notes & pitfalls

- New icons are frequently **unpublished**; `@figma-export` still exports them as long as they are real
  COMPONENT nodes on the configured pages and the name doesn't start with `.` or `_` (the
  `filterComponent` rule in `.figmaexportrc.mjs`). So an unpublished-but-correct component is fine.
- Don't confuse `COMPONENT` with an `INSTANCE` placed on the page — instances don't export.
- The export already strips `stroke`, `transform`, `color`, and `class`, but it does **not** strip
  `fill-opacity`. A shape carrying opacity will bake `fill-opacity` into the SVG; flag non-opaque
  fills if the repo's existing icons don't carry one.
- The component `description` field stores HTML-escaped text: `<` is read back as `&lt;` and `&`
  as `&amp;`. Writing a value that already contains `&lt;` or `&amp;` doubles the escaping (`&amp;lt;`).
  Decode the text to raw characters first (`&lt;`→`<`, `&gt;`→`>`, `&amp;`→`&`, `&nbsp;`→U+00A0), edit
  the raw text, and write it back. Verify after writing that no `&amp;` remains.
