# Destructive operations: read before deleting anything

**There is no undo, history, revert, or restore method anywhere in the `@framer/agent` API or the
CLI.** Verified 2026-08-01 across the full API surface. Once an API edit lands, the only exact
restore is the Framer editor's own Cmd+Z, and that degrades as further API edits pile onto the undo
stack. Whatever you delete, assume it is gone.

## Deleting a collection page also deletes its CMS detail page

Learned the hard way on 2026-08-01 in a template project. `removeNodes(["<id of /blog>"])`
removed **both** `/blog` and the CMS detail page `/blog/:slug`. The detail page was in scope to keep.

The trap is that the API actively suggests they are independent:

```js
const pages = await framer.getNodesWithType("WebPageNode")
for (const p of pages) console.log(p.path, await p.getParent())
// →  "/" parent=none · "/home" parent=none · "/blog" parent=none · "/blog/:slug" parent=none
```

Every page reports `parent=none`, including the detail page. **`getParent()` does not model the
collection-page to detail-page binding.** Passing that check is not evidence that a page can be
safely deleted.

Rules that follow from this:

- Before deleting any page, tell the user which *other* pages might go with it, and name them. A CMS
  detail page is invisible in the hierarchy but attached in practice.
- `serialize({ id, depth: 8 })` anything adjacent **before** the delete, not after. A captured
  structure is the difference between "recreate it in a minute" and "reconstruct it from memory".
- Deleting a page is not reversible through the API. Confirm it with the user every time, even when
  they approved the general cleanup. Approval of a plan is not approval of its collateral.

## Removal calls that do less than they say

- **`framer.removeNodes([ids])` can return `undefined` and remove nothing.** Three `FormSelectNode`s
  on a branch survived it twice; `DEL <id>` in `applyChanges` removed them. Read the parent back
  after any removal (2026-10-09).
- **`framer.removeNodes` silently skips component variants and variables**: it returns, and
  `serialize` still lists them. `DEL <variantId>;` and `DEL <variableId>;` through `applyChanges`
  remove them (2026-10-08).

## Overwriting a CMS item is just as final

`collection.addItems([{ id, fieldData }])` replaces every field it names **wholesale**: there is no
merge and no version history on CMS items. **Never use a live item as a test fixture**; make
throwaway edits on a field you have already copied, or not at all. If one does get clobbered, the recovery route is the **last published deployment**, which
still serves the old content until the next publish: fetch the live HTML and rebuild the item from it
*before* publishing anything else. Field-level detail is in [`cms.md`](cms.md).
