# CMS

- **`Collection.name` is a property, not `getName()`.** `getFields()` and `getItems()` *are* async
  methods. Item values come back on `item.fieldData[<fieldId>]`, keyed by field **id**, and an unset
  file field reads as `{"type":"file"}` with no `value`, a truthy object, so test `.value`.
- **Bind CMS fields only after the collection list exists.** A `text="var(--variable-<fieldId>)"` or
  `fill="var(--variable-<image>)"` binding in the *same* call that sets `collectionList.collection`
  either errors ("not in scope for the target") or, worse, renders the field's *name* ("Titre",
  "Extrait") on the canvas and the published page. Set `collectionList.*` first, then bind in a
  second `applyChanges`. Re-issuing the same `SET` fixes an already-broken binding. A binding with a
  transform also resets typography: bind first, style in a second `SET`.
- **A date field binds to text only through `toDateString`.** `text="var(--variable-<dateId>)"` is
  refused ("is of type date, expected string"); use `text.from="var(--variable-<dateId>)"
  text.transforms.0.name="toDateString" text.transforms.0.display="date-time"`. That `SET` drops the
  text style preset, so set `textStylePreset` again in a second `SET`. `dateStyle="long"` or
  `"full"` reports success and stays `medium` ("Oct 24, 2026, 7:00 PM"). The value is the typed
  wall-clock time written as UTC and reads the same in every time zone (2026-10-16).
- **CMS rich text is an HTML string.** A `formattedText` field's `value` is HTML, and
  `collection.addItems([{ id, fieldData: { <fieldId>: { type: "formattedText", value } } }])`
  updates it. Only the fields you name are touched, **but a named field is replaced wholesale, never
  merged.**
  - Framer normalises on write: `dir="auto"` is added, `<li>` becomes
    `<li data-preset-tag="p"><p>…</p></li>`, and a bare `<iframe>` pointing at YouTube or Vimeo is
    enriched into a real embed (Vimeo becomes Framer's own component).
  - **Code-block language is forced to `JSX` on write.** `class="language-bash"`,
    `data-language="shell"` and `language="bash"` all round-trip as `data-language="JSX"`. Write
    JavaScript or it renders mis-highlighted.
- **Reference fields take item *ids* on write but read back as *slugs*.** Passing slugs to `addItems`
  fails with `Bad reference, ID: <slug>`.
- **Option variable `cases`** take a JSON array string with escaped quotes.
- **Image fields accept a URL** through `$control__<id>.src`, but Framer fetches it server-side, and
  the whole `applyChanges` reports "One or more image updates failed" when one URL 404s (the text
  fields still land).
- **The server API (`framer-api`) writes only to user collections through `Collection.addItems`.**
  On a managed collection it throws `Expected the collection to be unmanaged`; use
  `ManagedCollection.addItems` on one the API created itself (`managedBy: "thisPlugin"`). There is
  no `framer.isAllowedTo` on the server API. It runs on Val Town/Deno via `npm:framer-api`. Items
  added this way reach the live site only at the next publish. 2026-09-25, `framer-api` 5.1.0.
- **To change some fields of a live item, use `item.setAttributes({ fieldData: { <fieldId>: { type,
  value } } })`.** It writes only the fields it names, unlike `addItems`. It returns `null` when the
  value did not change, so read the item back instead of treating `null` as a failure (2026-10-10).
- **Collection List sort cannot follow a variable.** `collectionList.sorting.0.direction` accepts
  only `asc` or `desc`. For visitor-chosen sorting, make one list per order (`DUPE` keeps the
  bindings) and bind each list's `visible` to a page variable with `convertFromString`, set by tab
  instances with `onClick.0.action="SET_VARIABLE_VALUE"` (2026-09-25).
- **Framer publishes every CMS field, bound or not.** Load-more pagination and other runtime reads
  fetch `framerusercontent.com/cms/<project>/<deployment>/<collectionId>-chunk-default-0.framercms`,
  a public file holding every field of every item. A field no page shows is still public. Keep
  private data (emails, member data) out of Framer CMS (2026-09-26).
- **Load-more pagination only works with Framer's built-in "Load More" component**, which
  `collectionList.pagination="load-more"` inserts into the list. A local look-alike (same variants
  and `onClick` event) renders but loads nothing, and a deleted built-in re-added by hand is not
  wired. Toggle `collectionList.pagination` to null and back to regenerate it. 2026-09-26.
- **`DEL <collectionNodeId>` deletes a collection and its items** (2026-10-06: gone from
  `getCollections()` and `getNodesOfTypes` afterwards). Older notes said the API could not; it can
  now, and there is no undo. Free a plan's collection slot this way only on the user's word. See
  [Destructive operations](destructive-operations.md).
- **Filter a collection list in the DSL.** On an option field:
  `collectionList.filters.0.transforms.0.name="equals"` with `value="<option name>"`. Six lists on
  one CMS, one per fundamental, each filtered by an option field, kept every row client-editable (2026-09-27).
- **Visitor filters run on page variables, and the native Select cannot drive a string field.**
  `contains` is case-insensitive, so it serves a search box. `FormSelectNode` binding is refused for
  `string` ("Expected boolean, collectionreference, controlReference"), and a collection synced by a plugin (from an applicant tracker, say) is all strings, so a dropdown on
  one needs a code component that writes the query parameter, or chips
  with `SET_VARIABLE_VALUE` (value `"null"` unsets). Checked on a published branch:
  `replaceState` plus a manual `popstate` updates the variable and the list; typing in the bound
  input writes the URL; `RESET_VARIABLE_VALUES` writes it with `pushState` then `replaceState`
  and fires no `popstate`, so code mirroring the URL must wrap those two methods (2026-10-09).
- **Group a filtered list under headings with one list per group.** Put the heading inside its list
  as a non-repeated first child with `visible.from="var(--variable-<listId>-item-count)"
  visible.transforms.0.name="greaterThan" visible.transforms.0.value="0"`: it renders once above
  the items and hides when filters empty the group. For one "nothing matches" message across all
  groups, add a list with every filter, hide its repeated item, and give it the empty state (2026-10-09).
- **Page variables are "not found" in a fresh session** until the page has been read: run
  `framer.agent.serialize({ id: "<webPageId>", depth: 0 })` before any `applyChanges` that binds
  `var(--variable-<pageVar>)` (2026-10-09).
- **Replace a list's whole sort in one value:** `SET <list> collectionList.sorting='[{"variable":
  "<id>","direction":"desc"}]'` (single quotes around the JSON). Per-index sets cannot remove an
  entry; `collectionList.sorting.2="null"` is refused (2026-10-09).
- **Prices wrap at the space before €.** Store them with U+202F (thin no-break space) as the
  thousands separator and U+00A0 before the currency sign (2026-09-27).

- **Show or hide a layer per CMS item with a computed `visible`:** `SET <id> visible.from="var(--variable-<fieldId>)"
  visible.transforms.0.name="isSet"`, adding `visible.transforms.1.name="negate"` for the opposite.
  Two rows in one list item, one shown per item, keep a single list in CMS order (2026-10-03).
- **A component's root cannot take `vh` heights** ("does not support viewport sizing"); a child
  frame can. A mobile menu scrim is a child `height="100vh"` under the drawer in the open variant,
  with the root's fill moved onto the bar and drawer so the scrim shows (2026-10-03).
- **`DUPE` of a component keeps its variable ids**, so the copy and the original share ids like
  `kXeHKbDdI`. Serialize only the component you mean to change in that exec before a
  `DEL <variableId>`, then read both components back (2026-10-03).
- **A text layer with a link variable gets a hand cursor even when the link is empty.** Keep a
  plain component without the link and a separate clickable one (2026-10-03).
- **New collection fields default to `required: true`.** `+Variable`, `+FileVariable` and
  `+OptionVariable` in a `CollectionNode` all came back required; an optional field needs a second
  `SET <fieldId> required="false"` (2026-10-06).
- **A video with a URL fallback from the CMS:** give the collection a `+FileVariable` (`allowedFileTypes`
  `[".mp4",".webm"]`) and a string `Video URL`, bind the `Video` component's `$control__file` and
  `$control__uRL` to them, and compute `$control__source` from the file:
  `$control__source.from="var(--variable-<file>)"`, transform 0 `isSet`, transform 1
  `convertFromBoolean` with `outputType="option" truthy="Upload" falsy="URL"`. `outputType="string"`
  is refused for an enum control. An uploaded file wins, an empty one falls back to the URL (2026-10-06).
- **A collection list item at `height="1fr"` inside an absolute, full-size list measured 200px.**
  Give the item `height="100%"` to fill (2026-10-06).
- **A field's type cannot change, and field names are title-cased.** To turn an empty link field
  into a string, with the user's approval, `DEL <fieldId>` and add `+Variable … type="string" scope="<collectionNodeId>"`.
  "YouTube link" came back as "You Tube Link"; pick a name that survives (here "Video"). Item
  values then go in as `$control__<snake_case_name>` (2026-10-03).

- **A `TextRun` cannot take `text="var(--variable-…)"`** ("Virtual rich text cannot use variable
  reference"). For a CMS-driven link with a label, use a small component with a Label variable and a
  Link variable, and pass `$control__link.href="/documentation/:slug"
  $control__link.collectionItem="var(--variable-<slug field id>)"`. A literal item id in `collectionItem` resolves to its slug (2026-10-08).
- **A rich text node bound to a `richtext` field wants `stylePresetHeading1` even when the content
  has no h1**, and the preset must carry `tag="h1"`; the linter repeats the warning until it is set (2026-10-08).
- **A component can hold a CMS collection list** (Docs Sidebar does), but add the repeated item's
  CMS-bound children only after `collectionList.*` is set, in a later `applyChanges`: in the same
  call the item's variables are "not in scope" and the binding is dropped while the node is kept (2026-10-08).
- **An `option` (enum) CMS field cannot feed a string component variable**: `toString` takes numbers
  only. Bind a string field instead (2026-10-08).
