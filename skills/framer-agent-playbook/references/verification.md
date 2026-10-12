# Verification: what lies to you

- **Screenshots come back as `image_url`**, a `data:` URL or an https URL. Screenshotting a
  `ComponentNode` fails with "viewport must be positive, got 0x0"; target a variant id. A breakpoint
  id returns the whole page at that width, which is the quickest responsive check (2026-09-22).
- **`getRect` returns the editor placeholder for unmeasured nodes**, `(-100,-100,200,200)` and
  sometimes `(-195..195)`, for Desktop nodes as well as replicas. Layout decisions made from it are
  garbage. Estimate width from the text: `chars × (fontSize × k + letterSpacing)` with `k` ≈ 0.53
  proportional, 0.60 uppercase, 0.62 mono. Treat a `ComponentInstanceNode` as about 320px, never 0.
  It only measures the page open in the editor: nodes on other pages return placeholders, and so
  does `$rect` in `serialize` output for them (2026-08-08). A freshly created replica breakpoint
  read 390×200 at negative coordinates while it was 390×7994 tall; take one unclipped screenshot,
  read its pixel height, then slice with `clip` (2026-07-31).
- **"Siblings too close" warnings on a breakpoint's direct children are stale.** Headless sessions
  report negative spacings of −200 to −700 from the same placeholder rects. Trust the screenshot.
- **In an exec script, `framer.screenshot(nodeId, { format, quality, scale, clip })` takes `scale` from 0.5, 1, 1.5,
  2, 3 and 4 only**; 0.55 or 0.8 throws. It returns `{ data: Buffer }`: write it to disk and read
  the file. It is `framer.screenshot`, not `framer.agent.screenshot` (2026-07-31).
- **A fresh screenshot URL does not mean a fresh render.** The `readProject` screenshot query returns a new URL on every
  call, even when nothing re-rendered. Identical file sizes across captures mean the change did not
  render; compare bytes before trusting a "no change" result (2026-08-07).
- **Canvas screenshots leave video blank.** Check video on the published site, or in its DOM (the
  `<video>` tag and the mp4 URL) (2026-07-31).
- **Stills show a fixed background in its first state only**, so scroll-driven background switches
  never appear in them (2026-09-15). An `onInView` appear above the fold can also look blank in a live screenshot
  taken right after load: that is the reveal mid-way, so check the DOM holds the content, and use
  `onMount` for the hero (2026-07-31).
- **Scroll-driven behaviour needs a real scroll on the published page.** Canvas screenshots and
  synthetic scroll lie in opposite directions: stills make each section's state look verified while
  the mechanism is untested, and `window.scrollTo({ behavior: "instant" })` or `scrollBy` loops do
  not trigger Framer's scroll-target detection, so a working `scrollVariantEffect` looks broken.
  Grepping the minified published bundle for `onScrollTarget` or `scrollVariant` proves nothing
  either. Scroll the page in a real browser, or ask the user to (2026-08-07).
- **Keep at most two WebGL instances on a page.** Three or four time out both the screenshot service
  and publish (2026-08-07).
- **Node screenshots render code overrides as on the live site, not as on the canvas.** An override
  that skips `RenderTarget.canvas` and hides a layer at scroll 0 hides it in the screenshot too,
  while scroll effects stay at their static values. To see what such a layer holds, save its
  `codeOverride`, `SET <id> codeOverride="null"`, screenshot, and set it back (2026-10-06).
- **Reads that look complete and are not.** Found auditing a large client file (2026-10-06).
  `getNodesOfTypes` ignores `pagePath` and returns matches from the whole project, so filter by
  `$scopeId` (the page or component id). Hidden layers carry `visible: "false"` as a string, so a
  `=== false` check misses all of them. One component variable with an empty name makes `serialize` on that component, and
  `getNodesOfTypes(["ComponentNode"])` for the whole project, throw `Expected a valid safe name for
  the variable ""`; `framer.agent.listComponents()` still lists every component. A `screenshot`
  query takes `id` (or `url`), not `nodeId`, and on a public URL returns the full page at 1200.
- **The linter cannot contrast-check text over an image fill** and says so. Screenshot it instead.
- **Framer's `place-content: center` on a stack also centres a block container.** Custom CSS that
  turned a section into `display:block` (to pin a sticky child) found the child 900px down until
  `align-content:normal` was added. Chrome applies `align-content` to block boxes now (2026-10-03).
- **Headless screenshots can draw a WebGL canvas inside a sticky frame off its real position.** Check
  with `getBoundingClientRect` before chasing a layout bug (2026-10-08).
