# Breakpoints

## Building a layout so breakpoints are cheap

Written after retrofitting Tablet and Phone onto 15 finished desktop pages
(2026-08-21). That job took roughly ten passes and four rounds of visual repair.
Almost none of it was necessary. The cost was set weeks earlier, when the desktop pages were built.

**The rule this document exists to state: the desktop build decides what every other breakpoint
costs.** Build desktop properly and a breakpoint is four decisions. Build it loosely and you pay for
it once per page, per breakpoint, forever.

### What a breakpoint should be

When the desktop layout is built on auto-layout, adapting it is almost entirely these four things:

1. **Turn some stacks vertical.** Rows of cards, rows of columns, side-by-side text and media.
2. **Drop grid column counts.** Four columns to two, two to one.
3. **Shrink the type.** Through text style presets, in one place, for the whole site.
4. **Scale padding and gap.** A 120px section rhythm on desktop is 60 on tablet and 40 on a phone.

Anything beyond that list is a bill for a shortcut taken on desktop. Not always avoidable, but always
worth noticing.

### How to build desktop so that stays true

**Type comes from text style presets, never from inline font sizes.** A preset carries its own
breakpoint sizes, so one edit resizes the whole site. An inline `fontSize` on a headline ignores the
preset system completely and has to be found and overridden individually. On those 15 pages,
115 text nodes carried inline sizes. Every one needed its own per-breakpoint override.

**Never put a `1fr` or `%` child inside an `auto` parent.** The child collapses to its minimum, which
for text means one character per line. It looks fine on desktop because the parent happens to be wide
enough. It is the single most common way a narrow breakpoint explodes: a paragraph that should be
three lines becomes 146 lines and a section grows to 8000px.

**Absolute positioning must never carry meaning.** A glow, a texture, a background: fine. A headline
overlaid on two columns, an annotation pinned beside an invoice row, a chart pinned under a hero:
those all break, because the coordinates were chosen for one width. If an overlap is part of the
design, the layout underneath still has to reserve room for it, so the overlay only ever adds polish.

**Do not set `height`, `minHeight`, or `height: 1fr` on a box that holds content.** All three survive
into every breakpoint. A 128px card floor that looks balanced in a seven-column grid becomes fourteen
tall empty boxes in one column. Grids stretch every row to the tallest one, so a grid needs `fit`
row height (`gridRowHeightType="fit"`) *and* no minimum on the cells before rows size to their own content.

**Make each row and card self-contained.** A row that can stand alone stacks cleanly. A row that
relies on a sibling's width, a shared baseline, or a divider drawn as a neighbour's left border
needs hand-holding. Side dividers in particular: when a row of columns becomes a stack, a left border
becomes a stray vertical line, and left padding becomes an indent. Either draw dividers as their own
elements, or expect to rotate them per breakpoint.

**Keep one layer per piece of content on every breakpoint.** A heading, paragraph, link or image
that exists twice, shown on desktop with a copy shown on tablet and phone, has to be edited twice,
and the site owner will change one and miss the other. Adapt the one layer instead: size, padding,
gap, direction, visibility, an instance's variant and its order within the same parent are all
breakpoint overrides. A breakpoint cannot move a layer to another parent, so when tablet needs an
element elsewhere, build desktop with it already in the parent both layouts share. Duplicate only
when no override gives a clean fallback, and name the copy after its breakpoints, such as
"Lead (tablet, phone)" (2026-10-13). A copy also costs in search: Framer publishes every breakpoint's
layers in the HTML and hides the others with CSS, so a copied heading ships twice, and a copied H1
gives crawlers a page with two H1s (2026-10-15).

**Name the things that cannot reflow.** A marquee and a dimensioned drawing are legitimate exceptions,
but only if they are recognisable. Call the marquee "Ticker" and the drawing "Plan". Then a
responsive pass can skip them instead of stacking a scrolling row of terms into a vertical list.

### Symptom to cause

| What you see at a narrow width | What desktop did |
|---|---|
| Text wrapping one character per line | a `1fr` or `%` child inside an `auto` parent |
| A headline still enormous after fixing presets | an inline `fontSize` on that node |
| Sections with tall empty areas | fixed `height`, `minHeight`, or `height: 1fr` on content boxes |
| Content overlapping other content | an absolutely positioned layer with real content in it |
| A row of logos as a one-per-line list | a row that should wrap, stacked instead |
| A scrolling ticker rendered as a vertical list | the marquee was not named, so a pass restructured it |
| Stray vertical lines and indents after stacking | dividers drawn as a neighbour's side border |
| A card wider than the screen with no override possible | a component whose variant root has a fixed width |

### Retrofitting, when it was not built that way

Order matters, because later passes recompute from desktop and overwrite earlier ones.

1. **Typography from presets**, once, for the whole site.
2. **Inline font sizes**, per node.
3. **Structure**: stack directions, grid columns, widths.
4. **Release constraints**: `height: 1fr`, fixed heights, `minHeight`.
5. **Repair collapse**: widen every `auto` parent that carries a flexible child, and give long
   children in a wrapping row their own full-width line. Run this **after** step 3, and run it in a
   loop until it reports nothing, because widening a parent can expose the next one.
6. **Density**: scale padding and gap.
7. **Hand-made per-page overrides, last.** Keep them in one file that always runs at the end. A
   generic pass computes from desktop and will silently undo a specific fix otherwise. This cost
   three rounds of rework on that retrofit: an option page's panel padding was set by hand, then a
   density pass recomputed it from desktop and put the overlap back.

### Verify on the published page

Canvas screenshots miss overlap and overflow that only the real CSS produces. Publish, then check
the live page at each width. Two checks find nearly everything:

- **Clipping**: for every element, compare its rect with its parent's. Flag it when the parent clips
  and the child spills. Page-level `scrollWidth` alone catches almost nothing, because the page root
  usually clips.
- **Collapse**: any paragraph or heading over ~20 characters rendering narrower than ~115px.

Exclude marquees and drawings by name from both, or they report as broken forever.

Framer-specific mechanics, including the mobile-first preset model and the properties that cannot be
overridden per breakpoint, are in [API traps](#api-traps) below.

## API traps

These are the API-level traps behind the method above, mostly from adding tablet and phone to 15
finished pages (2026-08-21).

- **Inline `fontSize` ignores preset breakpoints entirely.** A `RichTextNode` with its own
  `fontSize="118px"` renders at 118px on a phone. Sweep for inline sizes and override each per
  breakpoint, scaling px `lineHeight` and `letterSpacing` too.
- **Absolute pins take px only.** To centre one layer over another, make the parent a centred
  stack holding the front layer in flow, and pin the back layer absolute with `left="0px"
  right="0px" width="100%"` and a px `top`. An appear effect with
  `enter.opacity="1"` on a layer whose own opacity is 0.12 does fade it from 1 to 0.12 on load.
  `enter.scale` is absolute, not added: the default `1` on a layer scaled to 0.89 starts it 12%
  larger, so a glyph sized to clear the nav grew under it for the first second. Set
  `enter.scale` to the layer's own scale.
  `enter.rotateX` runs on the main thread while opacity runs on the compositor, so a stall during
  hydration (about 250 to 400ms, around 1.2s after load in headless Chromium) makes the fold jump
  from closed to open while the fade carries on. Start such an unfold after the page settles
  (delay 1.2s), and fold to `89` with opacity `0.01` rather than `90` and `0`, so the image is
  painted and decoded before it opens.
  For a photo that must open from its middle over something else (no page-coloured covers, no
  squash), use a component with variants instead of an appear effect: a clipping "Window" with no
  fill grows from 1px to full height in a centred stack while its image child moves from
  `top=-(h/2)` to `0` on the same transition, chained on load with `onAppear` → `SET_VARIANT`. On a
  replica variant, clear an inherited `onAppear` with `onAppear.0="null"` (`onAppear="null"` is
  ignored), or the last variant loops back (2026-10-06).
- **Reorder a layer on one breakpoint with `MOVE` on its replica:** `MOVE <replica-id><node-id>
  parent="<replica-id><parent-id>" index="n"`, where the parent is the layer's current one. The
  published page shows the new order on that breakpoint only. A `MOVE` on the primary moves the
  layer on every breakpoint, and no command moves it to another parent on one breakpoint. Read
  every breakpoint's order back after each write: an earlier per-breakpoint reorder was once lost
  after unrelated writes (2026-10-13).
- **Headings above a grid of cards cannot join the grid on desktop.** With `auto` rows every row
  grows to the tallest cell, so the heading row swells; with `fit` rows the cards lose their equal
  heights. Keep the heading row separate, and accept a tablet copy if tablet interleaves headings
  and cards (2026-10-13).
- **`position` is NOT overridable per breakpoint.** `SET <phone><id> position="relative"` is
  accepted, reads back as `relative` on the canvas, and is **silently ignored in the published
  CSS**. Re-pin the overlay instead (`bottom="0px"`) and pad the parent so nothing collides.
- **Component variant roots reject percentage widths** ("Percentage and fraction sizes are not
  supported on ground or variant-root nodes"), and the component's fixed width beats the instance's
  `width="1fr"`. `maxWidth="100%"` on the variant does not help. Give the variant a fixed px width
  that suits the narrowest use.
- **`minHeight` takes `0px`, not `auto`.** Grids stretch every row to the tallest;
  `gridRowHeightType="fit"` plus `minHeight="0px"` on the cells lets rows size to their content.
- **Navs.** Turning a nav row into a vertical stack produces a giant blob. Keep it horizontal, hide
  the links group, keep the wordmark and one CTA.
- **Technical drawings cannot reflow.** A free-layout frame placed by `left`/`top` gets
  `width="100%" overflow="auto"` on phone so it scrolls sideways at full size.
- **Applying `textStylePreset` to a node clears its inline type** (fontName, fontSize, colour),
  so a preset pass needs no detach step. A new `layout="grid"` defaults to `gridColumnCount` 2
  even with `gridColumnMinWidth` set; set the count (2026-09-26).
- **`minHeight` on a breakpoint is rejected** and the whole `SET` is dropped with it. Put it on a
  section inside the breakpoint (2026-09-25).
- **Equal-height cards in a grid: a `1fr` child inside a `1fr` item collapses the whole row to
  zero** (2026-09-27).
- **A 12-column grid with a big gap overflows.** Twelve columns at the default 50px minimum plus
  eleven 64px gaps is wider than 1120px, and the grid spills sideways. For a two-part split use a
  horizontal stack with a fixed-width column and a `1fr` column (2026-09-27).
- **A page-level fixed header cannot become a component with a drawer** when its layers carry
  scroll effects keyed to page markers (targets must be in the same scope). A `FixedOverlayNode`
  on the menu button works instead: an absolute panel pinned under the nav, rows with `link.href`
  plus `onTap.0.action="DISMISS_OVERLAY"`, a dismissible backdrop. Checked live on a phone: the
  row scrolls to its section and the overlay closes (2026-09-27).

- **A page that uses a layout template refuses `layout` and `fill` on its breakpoints.** Add a Main
  frame inside instead (2026-10-08).
