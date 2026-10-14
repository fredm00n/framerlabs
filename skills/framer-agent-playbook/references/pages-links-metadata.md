# Pages, links and site metadata

- **Template variables drive per-page chrome.** `+Variable v name="Current Page" type="string"
  initialValue="" scope="<layoutTemplateId>"` creates the variable but returns `{}` with no
  `renamedIds`; serialize the template to read its id and key (`$control__currentPage`). Bind the
  nav instance's control to `var(--variable-<id>)`, then `SET <webPageId> $control__currentPage="About"`
  on each page. A `visible` condition (`visible.from=... visible.transforms.0.name="equals"`) on a
  dot inside the nav shows the current page (2026-10-04).
- **A template variable switches an instance's variant only through a ComputedValue.** Binding
  `$control__variant="var(--variable-<id>)"` directly is refused ("Cannot assign variable directly
  to a variant control"). Map each option to a variant name with `convertFromOption`:
  `$control__variant.from="var(--variable-<id>)" $control__variant.transforms.0.name="convertFromOption"
  $control__variant.transforms.0.cases.0.from="Home" $control__variant.transforms.0.cases.0.to="Desktop Home"`
  (one case per option), then `.transforms.0.default="Desktop Home"` and
  `.transforms.0.outputType="option"`. `outputType="string"` is refused. Each breakpoint replica of
  the template needs its own binding, with its own variant names (2026-10-13).
- **A layout template breakpoint needs a fixed height.** `height="auto"` is refused ("Layout template
  breakpoints require a fixed pixel height"); the pages' own breakpoints take `auto` (2026-10-13).
- **`DEL <pageId>` does not delete a page.** `applyChanges` returns clean diagnostics and the page is
  still there. Pages come out only with **`framer.removeNodes([id])`**. `DEL` does work on ordinary
  nodes, so the failure is easy to miss in a mixed batch (2026-08-24).
- **The home page's path is immutable.** `SET <rootPageId> path="/x"` and
  `framer.setAttributes(id, { path })` both report success and change nothing (or error with "Cannot
  apply path to home pages"). There is no API for "make this other page the home page"; it is a UI
  action. Build new content inside the existing root page, or ask the user to reassign the home page.
- **A `+WebPageNode` with a taken `path`** still creates the page, under an auto path (`/page`), with
  all its children. Repair with `SET <page> path=…` instead of rebuilding.
- **An anchor link points at the node, not the `elementId` string.** Moving an `elementId` from a
  section to another frame broke every link to it (they went to `/`) until each `link.href` was set
  again, CMS link fields included (2026-09-27).
- **Anchors land at the very top of the viewport, under a fixed nav.** Framer has no scroll
  offset. Give the `elementId` to a 1px absolute frame at `top="-88px"` inside the section instead of
  to the section itself (2026-09-27).
- **The site *name* in the General tab still has no API** (2026-09-22).
- **`+WebPageNode <tmp> path="/404"` creates only the Desktop breakpoint.** Add Tablet and Phone
  with `CREATE_VARIANT`, then `name` and `width` in a later call (2026-09-27).

- **`metadata.noIndexSite="true"` on the home page ships as an `x-robots-tag: noindex` header**, not
  a meta tag. Check it with `curl -sI` (2026-10-08).
- **Published titles are cached.** A `curl` right after publishing can return the old root title;
  add a query string to see the new one (2026-10-08).
- **`link.current` needs the link on a `RichTextNode`**: `SET <label> link.href="var(--variable-<link
  var>)" linkStylePreset="<preset with link.current.*>"`. That works with a component's link
  variable, which a `TextRun` refuses. Take the link off the component root first (`link.href="null"`),
  or the linter reports nested links. A layout that must stay linked as a whole goes in a separate
  component; the nested link trap is in [Components and code](components-and-code.md) (2026-10-08).
- **No Flow effect on a layout template's breakpoint stacks.** On a page change Framer
  layout-animates their children, so a sticky nav that sat at the footer slides up from there
  (seen at 390 after a footer link; `transform: translateY(1658px)` easing to 0). Put Flow on each
  page's own stacks instead (2026-10-08).
- **Page effects run as a View Transition of the whole page.** `pageEffects.<page-id>.enter` and
  `.exit` animate `::view-transition-new(root)` and `::view-transition-old(root)`, so a layout
  template's nav and footer fade and move with the page content. Leave `x` and `y` at 0 when the
  template should stay still, and put the movement in `onMount` appear effects on the page's own
  content. The template's layers stay mounted across the change and keep animating under the
  crossfade: a template instance whose variant follows a page variable animates between the two
  pages' variants, with its variant transition (2026-10-13). The old side is a frozen snapshot of
  the whole page, template included, so a template layer that moves between the pages (a
  background light gliding behind the content) shows twice during the crossfade, frozen in the
  snapshot and live underneath. Visitors see it flicker and stall. For a moving template layer,
  give those pages no page effect and fade the page's own content in with appear effects: the
  template layer then animates alone (2026-10-14).
- **Rewriting links across a site:** CMS link fields take a new value through
  `item.setAttributes({ fieldData: { <fieldId>: { type: "link", value } } })`, which leaves the
  other fields alone. A search for links has to cover the layout template too: it is neither a
  `WebPageNode` nor a `ComponentNode` (2026-10-08). One reason to do it is a Framer affiliate tag:
  a partner link `framer.link/<tag>` redirects to `framer.com/?via=<tag>`, so append `?via=<tag>`
  (or `&via=<tag>`) to every framer.com URL (2026-10-09).
- **A component instance's link control refuses a path that has no page yet** ("No page matches
  that path"). When a build adds several pages that link to each other, create every page first,
  then set the menu and footer links (2026-10-14).
- **Smooth anchor scrolling is `link.smoothScroll="true"`** on each frame that carries a link, every
  variant included. With Lenis on the page, the jump to an anchor such as `/#plugins` eases over about 0.4s (2026-10-08).
- **The site language has no API.** `framer.getDefaultLocale()` reports `en-US` while the published
  `<html>` has no `lang`; set it in Site Settings (2026-10-08).
