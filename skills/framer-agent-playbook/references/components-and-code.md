# Components, code files and icons

- **A native accordion is built end to end through the agent**: the component, its two variants,
  the CMS binding and the click. The toggle is `SET <row> onTap.0.action="SET_VARIANT"
  onTap.0.controls.variant="cycle"`, set once on the primary variant; it propagates to the others.
  The generated `core-examples.md` has a worked FAQ accordion; read it before building one. The shape to
  build is in `framer-code-components-overrides` § Native First (2026-10-11).
- **Framer's built-in Ticker loops with one Web Animation** (`element.animate`, `iterations: Infinity`)
  whose duration comes from `speed`, so an override that changes `speed` restarts the loop and the
  strip jumps. To make it react to a swipe, drive the animation instead: find it with
  `wrapper.getAnimations({ subtree: true })`, set `currentTime` while dragging and `playbackRate`
  (negative runs it backwards) after. Its hover handler resets `playbackRate` to the hover factor.
  (2026-10-09)
- **A local harness can run Framer's own modules**: download the module URL from
  `framer.getNode(id).componentIdentifier` (`framerusercontent.com/modules/...`) and bundle it with
  the `framer` stub below. In cloud sessions launch Playwright with
  `/opt/pw-browsers/chromium_headless_shell-*/chrome-linux/headless_shell`; the full `chrome` binary
  refuses Playwright 1.48's old headless mode (2026-10-09).
- **Code files: the content is a property, not a getter.** `framer.getCodeFile(id)` and
  `framer.getCodeFiles()` both work. Read `.content` off the object; there is **no `getContent()`**,
  and calling it is what throws. Write with `codeFile.setFileContent(code)`, for example
  `const f = await framer.getCodeFile("Name.tsx"); await f.setFileContent(content);`.
- **`CodeFile.getVersions()` exists, so code-file writes are recoverable**, the one corner of the API
  with an undo. Canvas edits still have none.
- **`createCodeFile` returns `{}` on success**, and `createCodeFile` / `setFileContent` often report
  `waitForComponentLoader timeout` even when the write succeeded. Read `framer.getCodeFiles()` to get
  the `componentId` (`codeFile/<id>:default`) and to confirm the write.
- **Cross-file imports** (`./constants.tsx`) show as `Cannot find module` in `codeFile.typecheck()`.
  False positives; Framer's bundler resolves them at runtime.
- **`CodeFile.rename(newName)` re-slugs the published module URL** (`framer.com/m/<Name>-<hash>.js`)
  and **resets the file's path**, moving it out of any folder to the project root. On-canvas instances
  reference the file by id and survive, `$control__*` values included; remote consumers of the module
  URL do not.
- **`$control__<name>` only takes scalars.** Setting an array or array-of-objects control that way is
  silently dropped: the write reports success and the control stays `[]`. Use
  `framer.setAttributes(id, { controls: { ...node.controls, myArray: [...] } })`, spreading the
  existing `controls` so the others survive.
- **To switch a component's icon per instance, bind it to an icon variable.** `+IconVariable` on
  the component, then `SET <icon> $control__icon="var(--variable-<id>)"` on the primary variant (it
  propagates to every variant, so re-set any variant that needs its own icon, a Back button's Arrow
  Left for one), then `SET <instance> $control__<key>="Arrow Up Right"`. A `ComputedValue` on
  `$control__icon` (a boolean through `convertFromBoolean`) reports success and is dropped.
  `convertFromBoolean` cannot output a boolean either, so it cannot negate a `visible` binding, and
  `visible` with an `equals false` transform hid the layer in both states on the canvas (2026-10-11).
- **Link variables on instances are set with `$control__link.href="…"`** (2026-09-22).
- **A code component annotated `@framerSupportedLayoutWidth fixed` ignores the instance's `100%`.**
  It stayed at its intrinsic width inside a stretching wrapper. Use `any` for both axes when the
  component should fill a frame (2026-09-27).
- **A marketplace component id does not cross projects.** Reuse means re-adding it from the
  marketplace, or writing a code component.
- **Unused *external* components can only be removed in the UI.** They have no canvas nodes, so
  `removeNodes` has nothing to take. Find them by diffing the inventory's
  `### Current Project External Components` against a `ComponentInstanceNode` census.
- **The Logos icon set has marks only, no wordmarks.** Pair each mark with the brand name in a text
  node for a readable logo strip (2026-09-22).
- **Attaching a code override: the id is `codeFile/<fileId>:<exportName>`**, e.g.
  `SET <node> codeOverride="codeFile/a2AK5hc:withVisitorIp"`. Guesses like `<fileId>#<export>` or
  the bare export name are rejected. A file made with `createCodeFile` shows its exports in
  `framer.agent.listComponents` only after a new `session new`; before that the file lists `[]`
  and every `codeOverride` value fails (2026-09-25). On a client build (2026-09-29) the first
  `SET codeOverride` after the new session was rejected as "a code component export, not a code
  override", and the same command applied a minute later, while `listComponents` still listed `[]`.
  Retry once before rewriting the file.
- **Several `SET codeOverride` lines in one `applyChanges` apply only the first.** The call
  returns without an error and the other nodes keep no override. On 2026-09-30, eight photos in
  one batch, one got it; one SET per call worked for the rest, breakpoint copies included. Read
  every node back after attaching.
- **A code component's `ControlType.Font` value is a JSON string of a font selector**, not a text
  style: `$control__titleFont="{\"fontSelector\":\"GF;Rethink Sans-500\",\"fontSize\":\"32px\",
  \"letterSpacing\":[-0.02,\"em\"],\"lineHeight\":[1.2,\"em\"]}"`. Nothing binds it to a text style,
  so to match one, copy each breakpoint's values from `framer.getTextStyles()` (take `font.selector`)
  onto the instance and its breakpoint replicas. In `getTextStyles`, the base values are desktop
  and the `breakpoints` entry with `minWidth: 1200` holds the tablet values, `810` the phone ones
  (2026-09-29).
- **`DUPE` of a page section keeps only the desktop values.** Every tablet and phone override inside
  it (positions, sizes, padding) came back equal to desktop. Effect targets inside the copy are
  re-pointed to the copied markers, but element ids (`elementId`) are copied as is. Diff the
  breakpoint replicas of the original against the copy and re-apply (2026-09-30).
  The diff is scriptable: serialize `<breakpointId><sourceId>` and the desktop source at full depth,
  walk both trees in step, and turn every differing attribute into `SET <breakpointId><copyId>`,
  mapping ids through the `renamedIds` the `DUPE` returned. It re-applied 55 overrides across four
  sections in one pass (2026-10-05).
- **A `styleTransformEffect` section that leaves out a field gets 0.5 for it**, so a section setting
  only `opacity` came back with `scale: 0.5`, and one setting only `x`/`y` with `opacity: 0.5`. Write
  `x`, `y`, `scale`, `rotate` and `opacity` on every section (2026-09-30).
- **Native scroll effects cannot scrub width, height or radius**, only transforms and opacity. For a
  window that grows and shrinks with scroll, a code override that sets size from scroll is the
  shortest route; an override that reads its timing from marker layers stays editable on the
  canvas. The agent's node screenshots ran the override at scroll 0 and
  ignored scroll effects, so they show neither start nor end state.
- **Framer has no native effect that follows the cursor across the page.** Hover effects react only
  while the pointer is over the layer itself. A cursor parallax or magnet is one code override put on
  every layer: each measures its own centre against the pointer, so a falloff with distance makes
  the layers move by different amounts (2026-09-30).
- **An override that moves a layer which also has an appear or scroll effect should write the CSS
  `translate` property** (`el.style.translate = "4px 2px"`), not `x`/`y` or `transform`. Framer's
  effects own `transform`; `translate` composes with it, so both play. Gate pointer effects on
  `matchMedia("(pointer: fine)")` and `prefers-reduced-motion`, and skip `RenderTarget.canvas`.
- **Test an override before it touches a client file in a local harness:** esbuild bundles the
  `.tsx` with React, `framer-motion` and a two-line `framer` stub (`RenderTarget.current()` and
  `RenderTarget.canvas`), a page mimics the layer tree with `data-framer-name` attributes, and
  `playwright-core` on `/opt/pw-browsers` scrolls and moves the mouse while reading computed styles.
  Bundle with `--jsx=automatic`, since Framer files import no `React`. This proved a scroll-sized
  window override and a cursor parallax override before either reached the client file (2026-09-30).
- **`framer.getCodeFiles()` in the same exec call as `switchBranch` returns main's files.** Switch in
  one call and list in the next; branch-only code files then appear (2026-10-05).
- **Setting a component child on the primary and on a replica in one `applyChanges` call can land
  both values on the primary.** `SET qgIQzw4DC width=178px` then `SET B8nPrjmDCqgIQzw4DC width=103px`
  left every variant at 103 and the replica unchanged. Split them into two calls and read back
  (2026-09-29).
- **In an override on a component instance, pass `onClick`, not `onTap`.** Both reach the
  instance's root, but `onTap` is a pointer gesture: keyboard activation (Enter, Space) and a DOM
  `.click()` never fire it, so a button wired with `onTap` fails keyboard users and scripted tests
  alike (2026-09-26).
- **`SET <newVariant> name=...` in the same call as `CREATE_VARIANT` did not apply**, and a
  `$control__variant` computed value that names the variant then fails. Rename in a later call, or
  use variant ids as the `to` values.
  Rebinding a date text with a transform reset its typography again; restyle after.
- **`+ComponentInstanceNode component="<name>"` failed** for a component created earlier in the
  same session; its id worked (2026-09-25).
- **An override on a component instance can set `variant` by name** (`variant="Loading"`), which
  is how a code-driven state (loading, active) reuses variants designed on the canvas.
- **Built-in components with no style controls** (the pagination "Load More", for one) can be
  restyled from an override that injects CSS scoped to `[data-framer-name="<instance name>"]`.
  Name the instance first; published layer names survive (2026-09-26).
- **Component control keys are slugified from the variable name.** "Modalités" became
  `$control__modalit_s` and "Mention prix" `$control__mentionPrix`. A CMS field binds to an instance control as
  `$control__x="var(--variable-<fieldId>)"`. The Lucide icon set's `$control__color` takes a token
  var (2026-09-27).

- **A `FixedOverlayNode` must be the direct child of the node that opens it.** A second trigger
  cannot point at the first trigger's overlay ("must be a direct child"). `DUPE <overlay>
  parent="<other trigger>"` and wire each trigger to its own copy. Overlays cannot be screenshotted.
  Children of the overlay are absolutely placed: centre one with `centerAnchorX="50%"
  centerAnchorY="50%"` and `left`/`top` null (2026-10-03).
- **Overlays need `zIndex="10"`, the highest value Framer takes**, or a nav with a z-index draws
  above the backdrop. Overlays cannot be shared components, so put the content in a component
  (for example a "Media/Video Player" component: a Video string variable and a Close `EventHandlerVariable`) and
  give each overlay one instance with `onClose.0.action="DISMISS_OVERLAY"`. Framer's YouTube
  component inside it needs explicit size (`position="absolute"`, inset 0, or 100%): left on
  `auto` it renders as a small box in the top-left corner. A collection list item can hold its own
  overlay; bind the player's video to a CMS string field (2026-10-03).
- **Serialize a component in the same `exec` before binding its existing variables.** `SET <child>
  text="var(--variable-<componentVar>)"` from page scope fails with "Expected an existing variable"
  (and drops the text style) unless `framer.agent.serialize({ id: <componentId> })` ran earlier in
  that exec. Variables created in the same call bind fine, which hides the problem until later
  (2026-10-03).
- **A module URL goes in with `framer.addComponentInstance({ url, parentId })`**, for example a Lenis
  module from framer.com/m/…. Framer's own Smooth Scroll is not insertable (2026-10-03).
- **Plugin custom code is reachable:** `framer.setCustomCode({ html, location: "headEnd" })`,
  once per location, site-wide. Use it for CSS that must be right on first paint (2026-10-03).
  It is one unnamed entry per location (`headStart`, `headEnd`, `bodyStart`, `bodyEnd`), listed in
  the site's custom code settings under the plugin's name ("API Plugin" for the agent), so several
  snippets set this way end up in one block nobody can tell apart. The API cannot name an entry,
  add a second one, or read or edit the named snippets a user adds in the editor: `getCustomCode`
  returns only the plugin's own entry, with `disabled: true` once the user switches it off. When
  the user wants named snippets, hand them each snippet's code and placement to paste by hand,
  with Run set to "Once": "On every page visit" re-runs a script on each client-side page change,
  so an analytics snippet would initialise again (2026-10-12).
- **`+ComponentNode` creates its primary variant for you** (`"<tmp>-primary-variant"` in
  `renamedIds`). A `DUPE` aimed at the new component lands as a second ground node next to that
  variant, and the lint reports overlapping ground nodes. `MOVE` the copy's children into the primary
  variant, then `DEL` the empty copy. `DUPE` from any page or another component works as a source
  (2026-10-05).
- **`DUPE` of a node inside a non-primary variant does nothing and still reports `ok`.** Its id is
  the compound `<variantId><nodeId>`; `DUPE oQCsDRiDHk95nDgQoI parent=…` returned no error and no
  `renamedIds`, and nothing landed. `DUPE` the primary's id (`k95nDgQoI`) into the new component's
  primary variant instead: it lands as a child (no extra ground node), and its children keep the
  source component's variable bindings, so re-bind every `text` and `$control__` to the new
  component's variables (and re-apply `textStylePreset` with each `text`) (2026-10-08).
- **A shared block needs a page-only change: copy the component, swap the instance, hide the old
  one.** On a client's `/mission` page the closing block is `Image + Info` (6 pages) holding
  `Info Card` (more pages); adding a label next to its button meant `<Your name>/Info Card (Roles)`
  plus `<Your name>/Image + Info (Roles)` built from copies, a new instance placed after the old,
  and the old one `visible="false"`. Breakpoint sizes set on the old instance have to be set on the new one
  (2026-10-08).
- **An override on a component instance can set a component variable by its control key minus
  `$control__`**: `<Component {...props} cTA="View 59 open roles" />` changed the label of a
  `Button Rounded` instance whose control is `$control__cTA` (2026-10-08).
- **A code component's `$control__` keys come from the control titles, not the prop names.** A
  `srcUrl` prop titled "Video URL" reads back as `$control__videoURL`, and `autoPlay` titled
  "Autoplay" as `$control__autoplay`. The DSL accepted the prop names on create; read back with the
  title-based keys (2026-10-08).
- **The `<tmp>-primary-variant` id cannot be used in the call that creates the component.**
  `SET lcComp-primary-variant …` and every `+FrameNode … parent="lcComp-primary-variant"` in the
  same `applyChanges` failed with "target does not exist", while the `+Variable` lines in that call
  applied. Create the component and its variables first, then build the variant from `renamedIds`
  in a second call. A variable named "LinkedIn" was stored as "Linked In" (key
  `$control__linkedIn`) (2026-10-07).
- **An instance inside a `FixedOverlayNode` refuses `position="absolute"` without pins**, and the
  DSL cannot centre it. `framer.setAttributes(id, { position: "absolute", left: null, right: null,
  top: null, bottom: null, centerX: "50%", centerY: "50%" })` holds, on breakpoint replicas too
  (`<breakpointId><id>`). Pinning both `top` and `bottom` on an auto-height instance silently keeps
  only one; for a panel that must fit the screen, keep it centred and set `maxHeight="88vh"` with
  `overflow="auto"` on the component variant. `centerY` on a replica sticks only when `top` and
  `bottom` are nulled in the same `setAttributes` call (2026-10-07).
- **A component with click states needs its own stacked variant set for tablet and phone.** Components
  have no breakpoints, so `CREATE_VARIANT` one "Mobile - …" variant from each desktop variant, change
  layout on each, then override every tap per variant with the compound id:
  `SET <mobileVariant><itemId> onTap.0.action="SET_VARIANT" onTap.0.controls.variant="<mobileTarget>"`.
  Without the override, a tap in a mobile variant jumps back to the desktop layout (2026-10-05).

- **A coverflow from the native Slideshow:** `width`/`height` controls on `fit` stretched the one
  slide to the full instance width. What worked: controls on `fill`, `items` 1, the instance as
  wide as one slide (360×640), and `fadeOptions.overflow: true` so the neighbours show with
  `effectsOptions` (scale 0.82, rotate 12, opacity 0.5, perspective 1200). Negative
  `arrowPadding` is clamped, so arrows land on the neighbours; `arrowPosition: "bottom-mid"`,
  `arrowShouldSpace: false` and a negative `arrowPaddingBottom` put them under the slide. Set the
  nested objects with `setAttributes({ controls: { ...node.controls, … } })` (2026-10-06).
- **Slideshow and Carousel slot items are page ground nodes, so their size is the same at every
  breakpoint.** For different item sizes on phone, `DUPE` the ground nodes (overlays inside them
  come along and the `SHOW_OVERLAY` targets are re-pointed to the copies), slot them into a second
  instance, and toggle `visible` per breakpoint (2026-10-06).
- **The native Carousel's nested controls** are `sizingObject`, `snapObject` (`snapEdge`),
  `fadeObject`, `progressObject` and `arrowObject`, a different shape from the Slideshow's.
- **Neither the native Slideshow nor the Carousel iterates a CMS collection list.** A list in the
  slot becomes one slide holding every item (the Slideshow then shows overlapping copies of the
  whole row). For a CMS-fed Slideshow, slot one ground-node list per slide with
  `collectionList.limit="1"` and `collectionList.offset="<i>"`; the slide count is then fixed by the
  slots. The Slideshow also gives every slide the same main-axis width (`100%`, or a column share
  when `items` > 1), so mixed 16:9 and 9:16 items need a plain scrolling list instead (2026-10-06).
- **A horizontally scrolling frame needs `overflow="auto"` first.** With `overflow` visible,
  `overflowX="auto"` is refused ("Expected a value compatible with `overflow=\"visible\"`"); set
  `overflow="auto" overflowX="auto" overflowY="hidden"` together. Desktop mouse drag on such a row
  is not native: a code override adds it (click-and-drag with a short glide, scrollbar hidden, and
  a drag that ends on an item does not open it) (2026-10-06).
- **`pointerEvents` is refused on a component variant itself** ("Apply pointer events to a
  descendant") and drops the whole `SET` (2026-10-06).
- **The agent API cannot upload a font.** `getFonts` and `font-search` do not list custom fonts and
  there is no upload method; a client's licensed font goes in through the editor's Fonts panel,
  then the styles can switch to it (2026-10-06).
- **`+FixedOverlayNode parent="<variant>"` fails with "fixed overlays are not supported"** inside
  a component (2026-10-09).
- **Framer's built-in Video `startTime` is a percentage (0 to 100), not seconds.** The control is
  `{ title: "Start Time", min: 0, max: 100, unit: "%" }` in the published module. Name any
  variable bound to it with the unit. A hover variant that sets `$control__playing="true"` on the
  Video plays it on hover, and the primary's `false` pauses it again (2026-10-09).
- **A code override's `style` lands on the instance's root, which carries the variant name** as
  `data-framer-name`, not the instance name. Select it by `aria-label` or by the parent's name in
  tests (2026-10-09).
- **A hover or pressed variant cannot be screenshotted.** `readProject` answers "Variant node … must
  be a top-level variant". Check gesture states on the branch preview with Playwright instead
  (2026-10-08).
- **The DSL refuses percent pins on absolute children** (`left="15%"`: "Expected a pixel value").
  For a marker that must stay on the same spot of a scaling image, create it with pixel pins, then
  `framer.setAttributes(id, { position: "absolute", left: null, right: null, top: null, bottom:
  null, centerX: "15.99%", centerY: "32.7%" })`. Give the component a small fixed root (12x12) so
  the centre is the dot, and let the label and card overflow it (2026-10-08).
- **`height="fit-image"` with `width="1fr"`** keeps an image-filled frame at the image's ratio at
  every width, so percent-placed children stay registered to the picture (2026-10-08).
- **Framer hover variants open on tap on a touch screen** and close on a tap elsewhere, so a hover
  card needs no tap variant on tablet. In Playwright, drive a `hasTouch` context with
  `page.touchscreen.tap`; a `page.mouse.move` between taps left the later taps ignored (2026-10-08).
- **Remove an interaction with `SET <id> onMouseEnter.0="null"`.** `onMouseEnter="null"` and
  `onMouseEnter.0.action="null"` both report success and leave the handler in place. Variants that
  override the handler need the same `SET` on their compound ids (2026-10-08).
- **An icon next to uppercase Mono text sits low when both are centred in a stack**: the text's
  line box puts the caps above its centre. Align the row to the top and put the icon in a small
  `layout="null"` box with a negative `top` pin (15px icon at `-3px` beside 12px Mono with a 110%
  line height), then measure the ink on the preview (2026-10-08).
- **`framer.getTextStyles()` returns the last path segment as `name`** (`"S"`) and the full path in
  `path` (`"/Display/S"`). Find a style by `path`.
- **Framer lists an export as an override only when it is declared `export function withX(Component)`.**
  An export built by a factory (`export const withDrift8 = drift(8)`) is listed as a code
  component, and `codeOverride="codeFile/<id>:withDrift8"` fails with "is a code component export,
  not a code override". Write one declaration per variant around a shared hook. Read the file's
  `exports` after `setFileContent` to check each `type` (2026-10-09).
- **An override can move a native layer with motion values.** Return the props with
  `style: { ...props.style, x, y }`, where `x` and `y` come from `useSpring` and `useTransform`; an
  absolute layer keeps its pins and gains the transform. Wrapping the layer in a new `motion.div`
  instead breaks its absolute positioning (2026-10-09).
- **`blendingMode` layers blend with the page unless their parent is its own stacking context.** An
  absolute parent with `zIndex="0"` or a `masks.0.mask` gradient isolates them, as CSS `isolation`
  would; the published CSS carries both (2026-10-09).

- **`OptionVariable` cases are written `cases.0="canvas" cases.1="CMS"`.** A JSON array string is a
  parse error. Boolean, number, option and icon fields refuse `required` (2026-10-08).
- **An array-of-objects control cannot be set through `applyChanges`** ("unsupported array item
  type object"). Use a `ControlType.String` with `displayTextArea: true`, one entry per line
  (`Label = Value`), and parse it in the component; `\n` in the DSL value arrives as a newline.
  (2026-10-09, a filter-select component)
- **Bind an `+ArrayVariable`'s item fields** (`var(--variable-<itemFieldId>)`) in a later call; in the same call they are "not found", and the
  failed binding also drops the node's other attributes, so a caption silently fell back to Inter.
  An instance keeps its own copy of the array from the moment it is placed: changing the
  component's `initialValue` later does not reach it, and `applyChanges` cannot set an instance's
  array at all, so place instances after the defaults are final (2026-10-09).
- **A mouse-drag override on a row of image fills needs `onDragStartCapture={e => e.preventDefault()}`.**
  Framer renders fills as `<img>`; the browser starts its own image drag and cancels the pointer
  (2026-10-09).
- **A root link in any variant makes a link inside the component "nested" on publish**, even on an
  instance whose own variant has no root link (2026-10-09: "Nested link detected on layer Label
  due to parent layer Nav"). Give the inner-link layout its own component with no root link in any
  variant, and keep whole-row links in another. In one build, a doc nav link and a doc next link,
  and a contact row and a copy row, are split that way.
- **Framer's Copy Clipboard only renders text**, so it cannot be an icon that turns into "copied".
  A small code component that swaps the icon for a "copied" state on click does that (2026-10-08).
- **A gesture variant that drops a text link drops the `<a>` while hovered.** A nav link component's hover
  variant kept `linkStylePreset` but not `link.href`, so a hovered item was no link at all. With the
  link on a text layer, let the link style's `hover` do the hover and delete the hover variant (2026-10-08).
- **Framer's Copy Clipboard is in `framer.agent.listComponents()` under "additional"** (id
  `Hj20QU19p80mpYsvesiZ`). Its label and success text share one centred grid cell, so a shorter
  success text sits off-centre. In a monospace style, give both the same character count
  ("contact@yourstudio.com" and "copied, ready to paste" are both 22) (2026-10-08).
- **`codeFile.typecheck()` passes a file with `// @ts-nocheck` even when it cannot compile.** A
  duplicate `const` in one scope came back clean, then blocked publishing on every page with a bare
  `CodeError`. Build it locally first: `npx esbuild@0.23.0 File.tsx --loader:.tsx=tsx --format=esm` (2026-10-08).
- **Lenis eats the wheel over a nested scroller.** On a site with the Lenis component, an `overflow` auto frame does not scroll under the mouse until it carries `data-lenis-prevent`. Framer gives no way to add a data attribute, so an override sets it after mount (2026-10-08).
