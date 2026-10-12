# DSL and node attributes

- **Rotation is `rotation="-35deg"`.** `rotate="-35deg"` is ignored and the call still reports
  "Commands applied cleanly". A rotated, clipped frame with a counter-rotated pattern inside gives a
  diagonal cut (2026-10-06).
- **Changing `width` on an absolute layer in a breakpoint can re-pin it.** A right-pinned wall got
  `left="0px"` in the Tablet replica when only its width was set. Set `left="null" right="0px"`
  with the width (2026-10-06).

- **`stackDistribution="space-between"` publishes with no gap.** The stack keeps its `gap` value in
  the editor, but the published flex box gets `gap: normal`, so in an `auto`-height stack (no
  leftover space) the children touch. A card's links sat flush under its summary on phone. Use
  `start` with a `gap` wherever the stack hugs its content, and keep `space-between` for a stack
  with a real height to fill (2026-10-11).
- **Send one `MOVE` or `SET` per `applyChanges` call.** Several `MOVE` lines in one call reported
  "applied cleanly" but ran only the first, and put that node in the *last* line's parent. Two
  `SET` lines on different nodes applied only the first too (see also the `codeOverride` note).
  Read every node back after each change (2026-10-09).
- **A hairline of photo under a page-coloured cover is a layer seam, not a border.** Covers
  animated with transforms are their own GPU layers, and on fractional positions the photo's edge
  row shows past them (seen in Chrome on Windows and headless Chromium at 1x). Give each cover a
  page-coloured `boxShadows.0="0px 2px 0px 2px <token>"` toward the open edges, inside a frame
  with overflow clip (2026-10-09).
- **Corner radius is `radius` and a frame's shadow is `boxShadows.<i>="x y blur spread color"`.**
  `borderRadius` and `boxShadow` are dropped without an error, and `shadows.<i>` is refused on a
  `FrameNode` (2026-10-10).
- **`framer.setAttributes(id, {...})` reaches raw node properties the DSL does not expose**, notably
  `locked` (`constraintsLocked` is a different, positional thing) plus anything else
  `framer.getNode(id)` returns. Try it whenever a property looks missing.
- **A rejected attribute fails the whole `SET`**, not just that attribute. `fontName="null"` or
  `fontSize="null"` on a text node with a preset is rejected ("Inline styles are ignored while a text
  style preset is set"), and a `textStylePreset` in the same `SET` is dropped with it.
- **Event handlers:**
  - On a non-primary variant, `onTap="null"` does not remove a handler inherited from the
    primary: the read-back still shows it. Override it with a tap that targets the variant itself (2026-10-05).
  - A variant override equal to the primary's current value is not stored. Setting `height="auto"` on
    variants while the primary was also `auto`, then changing the primary to `800px`, moved them all
    to 800px. Change the primary first, then override the variants (2026-10-05).
  - To cross-fade content between variants, keep every block visible in every variant and hide the
    inactive ones with `position="absolute" left="0px" top="0px" width="100%" opacity="0"
    pointerEvents="none"`; the variant transition animates the opacity, where a `visible` swap is
    instant. Other variant sets (mobile, tablet) inherit the primary's new absolute/opacity state, so
    re-set them after the primary: `visible="false"` on inactive blocks, `relative` and opacity 1 on
    the active one (2026-10-05).
  - A link inside a component can target a page anchor if the href carries the page path:
    `link.href="/platform#features-design"` works, while `"#features-design"` is refused ("Hash links
    without a path are only supported for nodes inside web pages"). A link and an `onTap` can sit on
    the same layer. Refusals like this come back in the result's `errors`, not as a thrown error, so
    print `r.errors` after each call (2026-10-05).
  - Auto-advancing loop: `SET <variant> onAppear.0.action="SET_VARIANT"
    onAppear.0.controls.variant="<next>" onAppear.0.delay="6s"` on each variant, the last pointing
    back to the first. A variant without `onAppear` stops the loop (2026-10-05).
- **Any `hoverEffect.*` silently adds `scale: 1.1`.** It is invisible in the response and shows only
  in a serialize. Pass `hoverEffect.scale="1"` on every hover (2026-09-17).
- **`appearEffect` with `trigger="onInView"` replays by default.** Elements fade out when scrolled
  past and fade in again on the way back. Set `appearEffect.replay="false"` on every one and add no
  `exit` state (`onMount` effects run once anyway, and `replay` on them only raises a warning) (2026-09-22).
- **`aspectRatio` needs a pixel `height` next to it**, even when the width drives the result.
  `width="100%" height="504px" aspectRatio="2.2222"` renders at the ratio; without the height the
  command is refused, and `width="1fr" height="auto" aspectRatio="1"` is refused too.
- **`height="1fr"` on a `RichTextNode` collapses it** to one line and the text spills over its
  siblings. To push a card footer down, add an empty spacer frame with `layout="stack" width="1fr"
  height="1fr" minHeight="0px"`.
- **Absolute children pinned `top="0" right="0"` inside a stack parent straddle the corner.** A
  14px inset lands them inside.
- **A parent's filter cascades onto blend-mode children.** `grayscale="100%"` on an image frame also
  desaturates a `blendingMode="color"` tint layer inside it, and every button in that subtree. For a
  duotone, drop the filter and let the colour blend do the conversion (2026-08-24).
- **`DUPE` works on a whole `WebPageNode`**: the full page tree copies in one command. The page's
  display name resets when you later `SET` its `path`, and the copy still references the source's
  tokens and presets. Restyle by creating a parallel token set and re-pointing, never by editing the
  shared originals.
- **Enumerating node types uses two different vocabularies.** `framer.getNodesWithType(type)`
  accepts only canvas types and **rejects `ExternalModuleNode`** (its error lists the accepted ones).
  `framer.agent.getNodesOfTypes({types})` takes the style-preset types (`LinkStylePresetNode`,
  `InlineCodeStylePresetNode`, `BlockquoteStylePresetNode`, `TableStylePresetNode`,
  `ImageStylePresetNode`, `TextStylePresetNode`) and is the only way to discover their ids.
- **`scrollsectionref` variables can be neither created nor written by the agent** (`+Variable`
  rejects the type; `SET` reports "unsupported type"). Scroll-target indirection through page-level
  variables has to be wired by hand. Setting `scrollVariantEffect.sections.<i>.target` **directly**
  on an instance is in the DSL and works.
- **Read `scrollVariantEffect` targets back after setting them.** Targets set with temp ids from the
  same batch were stored as those names, and the published page shipped with no sections, so the
  effect did nothing. Re-`SET` them with real node ids, after the targets carry
  `scrollTargetEnabled` and `elementId` (2026-09-27).
- **The session that set a `scrollVariantEffect` shows its targets under their temp ids.** Even when
  real ids were passed, `serialize` in the same session printed `prImg0` and `piOpen`. A second
  session (`session new`, then switch to the branch) read the real ids back, so the stored value was
  right. Do the read-back from a fresh session (2026-09-28).
- **The DSL rejects a negative cubic-bezier value in a transition.** Existing nodes serialize with
  `tween 0.7,-0.35,0.3,1.1 1s 0s`, and passing that same string back fails with "<ease> could not be
  parsed". `tween 0.44,0,0.56,1 1s 0s` works, and `DUPE` keeps the original curve (2026-09-28).
- **Slot controls on a code component are DSL-only.** `framer.getNode(id).controls` does not list a
  slot such as a slideshow's `content`, and `setAttributes` with it does nothing. Set
  `$control__content.<i>="<id>"` in the DSL, and clear leftovers with `="null"`. To reuse a
  configured slideshow, `DUPE` the instance and re-point its slot (2026-09-28).
- **A marker-style highlighter under a text preset is a link style.** Inline `textDecoration` on a
  run is dropped when a preset is set, and `textBackgroundColor` covers the whole line height. A
  `LinkStylePresetNode` with a thick underline and a negative offset on a linked run works, but only
  with `link.textDecorationSkipInk="none"`; skip-ink hides a thick underline entirely (2026-09-27).
- **Chrome's focus ring on a published link traces every child box, hidden ones included.** A
  logo link holding opacity-0 hover layers (handles, a size tag) showed a ring around the logo plus
  an empty box where the tag sits. Fix it in the site's head custom code with
  `a[name="<instance name>"]:focus-visible { outline: none }` and a focus state that shows the hover
  layers by `[data-framer-name]` (2026-10-10).
- **Framer draws a layer's border with `::after` on `[data-border="true"]`.** Custom CSS that puts
  its own `::after` on a published layer (a focus ring, a badge) replaces that border and inherits
  its sizing. Use `::before`, which no Framer layer used on the site checked (four pages, 2026-10-10).
- **Fluid absolute positioning needs the raw API and a non-stack parent.** The DSL rejects `%` pins.
  `framer.setAttributes(id, { position: "absolute", left: null, right: null, top: null, bottom: null,
  centerX: "40%", centerY: "55%", width: "30%", height: "60%" })` holds, but only when the parent is
  not a stack; absolute children of a stack parent need px pins. For shapes that must scale with a
  word, make each letter a relative child of a horizontal stack with a `%` width, and pin its pieces
  by `%` centre inside it (2026-09-27).
- **A per-breakpoint `SET` of `styleTransformEffect.sections.<k>.x` works**, so tablet can carry its
  own smaller offsets (2026-09-27).
- **A static `opacity="0"` wins over the effect on the published site.** The canvas preview fades
  the node in, the live page never shows it. Leave the static value at 1 and let the effect's first
  section start at 0 (2026-09-27).
- **`position` cannot change per breakpoint.** To have a sticky column on desktop only, keep the
  child sticky and wrap it in a column frame whose height is `1fr` (Fill) on desktop and `auto` on
  tablet and phone. A `100%` (relative) wrapper does not let the child stick; it has to be Fill
  (2026-09-27, 2026-09-28).
- **A tall sticky header over a filtered list hides the list.** A sticky element stays until the
  bottom of its parent, so the last rows of the list beside it slide under it: with a 240px header
  the list looks empty at its end, and a filter that shrinks the list while the header is stuck
  leaves every match under it. There are no margin attributes to fix it natively. A code
  override on the list fixes it: it gives the list a negative bottom margin the header's height, pads
  the outer frame by the same amount, and scrolls back to the list top when a filter changes
  (2026-10-09).
- **Never find an override's own element by React `useId`.** On a published page that hydrates
  with a tree different from the server's, the DOM keeps the server's id while the code queries the
  client's, so `querySelector('[data-x="' + useId() + '"]')` finds nothing and the override goes
  dead without an error. It worked on one publish and broke on the next. Make the id in the browser
  after mount (`useState(null)` plus a `useEffect` that sets a random string) and render the
  attribute only once it exists (2026-10-09).
- **A frame's children can be `display: contents` wrappers.** Framer wrapped each component instance
  in a frame in one, so `children[i]` had no width, no `offsetLeft` and no `offsetParent`, and
  `scroll-snap-align` on it was ignored. Step through to the first child that has a box, and measure
  with `getBoundingClientRect`, not `offsetLeft` (2026-10-09).
- **Images in a sideways-scrolling row pop in late.** Framer renders every image with
  `loading="lazy"`, and the browser only fetches an image in a horizontal overflow row once it
  scrolls into view, so each slide appears a moment after it arrives. Set `img.loading = "eager"` on
  the row's images once the row nears the viewport (an `IntersectionObserver` with a 1000px
  `rootMargin`), so the page's first load is not affected (2026-10-09).
- **Never clear an inherited border on a breakpoint with `border="null"`.** The canvas shows no
  border, but the published page drew a 1px top and bottom border on that node at tablet and phone.
  `borderLeft="0px solid"` fails with "Border is incomplete". What worked: keep the desktop widths
  and set `borderColor="rgba(0,0,0,0)"` on the breakpoint. Check borders live, not on the canvas (2026-09-27).
- **Virtual rich-text child ids (`v:<nodeId>:0:0`) go stale between exec calls**, and a `SET` on
  one fails with "no longer exists". `framer.agent.replaceText({ id, searchText, replaceText })` on
  the owning node works and keeps the preset. A `TextLineBreak` adds no space to `textContent`, so
  end the first run with a space or the H1 reads "pourun" to crawlers (2026-09-27).
  Reading the owning node with `serializeNodes` earlier in the same exec script also makes the
  `SET` on its `v:` ids work (2026-09-27).
  A `SET` on any run also renews every `v:` id of that node, so the next `SET` on a sibling run
  fails the same way: serialize the node again before each `SET` (2026-10-10).
- **A fixed header covers anything that scrolls under it**, including layers animated up into it.
  To let flying letters land on top of a fixed nav, give their section a higher `zIndex` than the
  header, `pointerEvents="none"` on that section, and `pointerEvents="auto"` on its interactive
  children (2026-09-27).
- **A window that grows from inline to full screen is one frame with a different size per variant.**
  It can sit in the text row as a relative stack child in one variant and be `position="absolute"`
  pinned to all four edges in the next; `CREATE_VARIANT` plus `SET` accepts both. Chain the load
  steps with `onAppear` delays and hand over to `scrollVariantEffect` with markers placed so none is
  50% visible at scroll 0, or the effect overrides the load chain. Absolute children centred with
  `centerAnchorX/Y` and null pins cannot be written in the DSL (it wants a pin); use
  `framer.setAttributes(id, { position: "absolute", left: null, right: null, top: null, bottom: null,
  centerX: "50%", centerY: "50%" })` (2026-09-29).
- **`styleTransformEffect="null"` on a replica turns an effect off for that breakpoint.** Effects
  keyed to markers that are hidden on tablet and phone would otherwise hold their first section,
  which can mean invisible (2026-09-27).

- **Preset-controlled properties are refused on a text node that has a `textStylePreset`**
  (`lineHeight`, `textDecoration` and the like), and the refusal kills the whole `SET`. `fontWeight` on
  a `TextRun` is the exception and goes through, which is how a light "&" sits inside a heavier line (2026-10-03).
- **The text style decides the tag.** A `+TextBlock … tag="h1"` under a preset whose tag is `p`
  publishes as a `<p>`. Make a style with `tag="h1"` (for example `Heading/H1` and `Utility/H1 Hidden`)
  and read the published DOM for `h1` (2026-10-03).
- **Percentages are dropped silently on pins and appear offsets.** `top="100%"`/`bottom="100%"`
  came back `null` and `appearEffect.enter.y="100%"` came back as `100` (px). For a reveal that
  must scale with the box, fold the cover instead: `rotateX="90deg"` at rest with
  `transformOriginY="0%"` (or `100%`) and `appearEffect.enter.rotateX="-90"`. **Appear values are
  added to the layer's own transform**, so `enter.rotateX="0"` starts at the rest state and nothing
  moves; and a layer that starts folded has no height, so `onInView` never fires for it. Set
  `appearEffect.enter.opacity="1"` too: the enter opacity defaults to 0. Check on the published
  page that the layer's first inline `transform` is the start state, not the rest state (2026-10-03).
- **A layer's own `rotation` is dropped from its appear start state.** A band set at
  `rotation="-13deg"`: with `appearEffect.enter.x="-40"` the published start transform was
  `translateX(-40px)` with no rotate, so the band swung from 0 to -13 degrees on load. Restate
  it: `appearEffect.enter.rotate="-13"`, per breakpoint when the rotation changes there
  (2026-10-06). (This differs from `rotateX` above, which is added.)
- **Gesture variant names cannot be set**; hover variants stay "Variant" (2026-10-03).
- **A component variant refuses `tapEffect`.** Put the fill, padding and press effect on an inner
  "Surface" frame and keep the variant itself bare (a `CTA/Button` component, for example) (2026-10-04).
- **A text node takes no padding.** To line a 10.5px mono number up with a 17px question beside it,
  give the number its own style whose line height equals the question's in px (`Label/Form Number`,
  23px). `lineHeight` on the node is refused while a style is set (2026-10-04).
- **`aspectRatio="null"` is refused.** `height="auto"` alone drops the ratio (2026-10-04).
  `framer.setAttributes(id, { aspectRatio: null })` clears it outright (2026-10-05).
- **A one-value `padding="0px"` on a breakpoint replica left the inherited `0px 80px 0px 80px` in
  place**, with no error. Write all four sides (`padding="0px 0px 0px 0px"`) (2026-10-05).
- **Four `1fr` columns to two or one: wrap the stack, not a grid.** Switching a tablet replica to
  `layout="grid"` collapsed the text inside each `1fr` item to one character per line. On the
  replica, `stackWrapEnabled="true"` plus `minWidth="300px"` on each `1fr` child gives two columns
  at 810; on phone, `stackDirection="vertical"` (2026-10-05).
- **An empty `$control__x=""` falls back to the component's default text**, and
  `SET <varId> initialValue=""` does not change a component variable. Bind the control to a CMS
  field, or hide the layer with a visibility condition (2026-10-04).
- **A collapsed accordion panel keeps its padding at height 0.** Set the Closed variant's panel
  padding to 0 as well as its height, or every closed row is taller than it looks (2026-10-04).

- **A `BooleanShapeNode` (a vector logo) cannot be resized through the API.** `width` on it is
  refused by the DSL ("Cannot apply `width`") and `framer.setAttributes` accepts it and changes
  nothing. Either `DUPE` a copy that already has the size you need (another page's larger logo
  works as a source) or wrap it in a frame and set `scale` on the frame, per breakpoint (2026-10-06).
- **A `styleTransformEffect` scale multiplies the layer's static `scale`.** A wrapper at static
  `scale="2.6"` with an effect section at 2.6 rendered at 6.76. Keep the static value at 1 and
  let the effect sections carry the size (2026-10-06).
- **Never scale a layer up with an effect; give it its largest size and scale down.** A 318px
  vector logo scaled 1.8× by a `styleTransformEffect` looked sharp on load and soft after scrolling
  away and back: the browser rasterises the transformed layer at its layout size. For a large
  vector, `framer.exportSVG(id)` the shape, give the SVG a `viewBox` and the size you need, upload
  it with `framer.uploadImage({ image: "data:image/svg+xml;base64,…" })` (the `{ bytes }` form was
  refused in exec) and use the URL as a frame `fill`. `framer.addSVG` failed with "Failed to
  optimize SVG" even on a plain rectangle, and `removeNodes` on a `BooleanShapeNode` returned
  without removing it (2026-10-06).
- **`onScrollTarget` targets must be scroll targets.** Plain marker frames are rejected with "Every
  target section must be a scroll target in the same scope"; give each
  `scrollTargetEnabled="true"` and an `elementId` first (2026-10-06).

- **A component instance pinned left/right/top/bottom stays fit-content.** For an overlay that fills
  its parent, set `position="absolute" left="0px" top="0px" width="100%" height="100%"` (2026-10-08).
- **`ariaLabel` labels an empty link frame** (a card's Hit Area). Binding it to a component variable
  needs a `serialize` of the component earlier in the same exec. An ASCII-art logo link needs one
  too, or its accessible name is the box drawing (2026-10-08).
- **A layout template's nav root with `overflow="clip"` cuts anything that sticks out of it**, such as
  a hover selection on the logo at the nav's left edge. The desktop variant can be `visible`; the
  phone menu keeps its clip (2026-10-08).
- **`overflowY="auto"` and `overscroll` are refused until `overflow="auto"` is set**, in an earlier command (2026-10-08).
- **A fixed frame pinned left, right, top and bottom publishes stretched** to those pins, though `serialize` still reports `width` and `height` as 100px (2026-10-08).
