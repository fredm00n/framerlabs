# Text styles and fonts

- **Merging text styles: create the target, re-point every layer, then delete.** Re-pointing a
  layer with `SET <id> textStylePreset="<new>"` keeps its inline colours on the root and on each
  run, and breakpoint replicas follow the primary. Component variant overrides do not: serialize
  every layer that used the old style (primaries, replicas, variants), read back which still name
  it and `SET` those too. Where the old and new styles differ in colour, compare each run's
  effective colour before and after on every copy, variants included: a colour fixed on the
  primary flows into a variant that already had the right colour from its own style. Rename by
  the same route; renaming a style in place orphans its references (2026-10-05).
- **`getNodesOfTypes({types:["TextStylePresetNode"]})` also returns two breakpoint copies of every
  style**, named without the folder and carrying `$originalId`. Count and delete only entries
  without `$originalId`; copies whose original is gone can be deleted the same way.
- **`readComponentControls` keeps a new component's variant names from creation** ("Variant 2")
  after `SET name` renames them. Instances still resolve `$control__variant="<new name>"`; check
  the published page rather than the controls list.

- **Text style presets are referenced by full path, colour styles by id.** A node stores
  `textStylePreset="Display/XL"`, so foldering a text preset renames it and orphans every reference.
  Colour styles are `var(--token-<id>)` and reorganise freely. Component names take folders safely
  too, because instances reference `canvasComponent/<id>:default`.
- **A preset blocks inline type.** `+RichTextNode … textStylePreset="X" fontSize="14px"` (or
  `fontWeight`) errors with "Inline styles are ignored while a text style preset is set". Inline
  `textColor` and `textAlignment` are accepted, so colour overrides on dark sections need no extra
  preset. To override type per node, detach first with `SET id textStylePreset="null";`, then a
  **second** `SET` with the inline styles.
- **Preset breakpoint slots do not map to fixed widths. Read `minWidth` back after writing.** Two
  projects gave opposite orders:
  - One project (2026-08-21, presets that existed before tablet and phone were added):
    `default` was the narrowest slot (`minWidth: 0`), then `small` (810), then `medium` (1200).
    Setting `minWidth` yourself errored ("Cannot set a positive `minWidth` on the last/narrowest
    breakpoint slot"); writing the slots in two calls (`default` + `medium`, then `small`) let
    Framer assign them.
  - Another (2026-09-22, presets created on a single-breakpoint page): `default` was desktop
    (`minWidth: 1200px`, not settable), `medium` 810px, `small` 0px, and a `large` slot was rejected.
    One `SET` with all three slots worked.

  So: write the sizes, `serialize` the preset with `attributeFilter: ["breakpoint"]`, and check that
  the biggest `fontSize` sits on the biggest `minWidth` before styling anything else.
- **`letterSpacing` on a preset takes px or rem only**, while `breakpoint.<label>.letterSpacing`
  accepts em.
- **A failed `text=` binding drops the node's preset** (it falls back to Inter 16 black). So does a
  plain `SET <id> text="…"` on an existing preset-bound node. Re-apply `textStylePreset` in the same
  or the next `SET`.
- **`ControlType.Font` ignores `fontWeight` in `defaultValue` and rewrites unknown families.** It
  always hands back `fontWeight: 400`, and a `fontFamily` it does not recognise becomes Inter. Size,
  letter-spacing and line-height stick. Font control values set through the DSL are stored but do not
  render until the control is touched in the UI, so make the component's own defaults match the
  intended families.
- **Code-component colour controls do not resolve `var(--token-…)` strings.** Pass literal hex to
  `$control__` colour props of code components.
- **Text presets carry OpenType features by default** (`blwf`, `cv03`, `cv04`, `cv09`, `cv11`).
  On Mona Sans, `tnum` draws a dotted zero; set it `off` on presets that show prices or counts (2026-09-27).
- **A hover underline under a preset needs its own preset.** Inline `textDecoration` is rejected on
  a preset-bound node, so make "Title" and "Title Hover" presets and swap them between variants (2026-09-27).
- **Link presets need `link.textColor`**, or linked text renders browser blue on the live site (2026-09-27).

## Check a font's variation axes before designing around them

Framer's copy of a Google font does **not** always carry the same variation axes as the version on
Google Fonts. Verified 2026-08-01: **Framer's `Archivo` exposes only `wght` (100–900), no `wdth`**,
even though Google's Archivo has a width axis. A design built on wide Archivo renders silently at
default width in Framer; nothing errors, it just comes out narrow.

Check first, with `name` for a specific family:

```js
await framer.agent.readProject([{ type: "font-search", name: "Archivo" }], { pagePath: "/" })
// → { name, weights, styles, category, variationAxes: [{tag,minValue,maxValue,defaultValue}], openTypeFeatures }
```

- **`{ query: "..." }` returns `[]` for a bare family name**, including fonts that definitely
  exist. An empty result is not proof of absence. `name` also returns `[]` for
  a family that is already in `<project-fonts>` (Mona Sans did), and for workspace custom
  fonts (Departure Mono did).
- Families with a usable `wdth` above 100 (as of 2026-08-01): Mona Sans and Hubot Sans (75–125),
  Georama (62.5–150), Saira (50–125), Science Gothic (50–200), Google Sans Flex and Roboto Flex
  (25–151). `Archivo Narrow` and `Archivo Black` are separate families, not axis positions.

`framer.getFonts()` returns all ~9,500 individual cuts with a `selector` like `GF;Mona Sans-700`
(`GF` = Google Fonts, `FS` = Framer's own). It confirms a family exists and which cuts ship, but it
does **not** report variation axes. Only `font-search` does.

- **Text style presets start with `paragraphSpacing` 20.** A multi-line ASCII logo or file tree
  shows gaps between lines until the preset gets `paragraphSpacing="0"` (2026-10-08).
- **`openTypeFontFeatures.*` on a node that has a text style fails the whole `+RichTextNode`.**
  Put the feature on the preset or drop it (2026-10-08).
