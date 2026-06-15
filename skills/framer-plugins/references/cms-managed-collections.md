# Framer CMS — ManagedCollection Reference

Everything specific to CMS plugins: the ManagedCollection API, the sync algorithm,
field types, and CMS-only pitfalls. Skip this file entirely for canvas / image /
other non-CMS plugins. For the general SDK, see [SKILL.md](../SKILL.md) and
[api-reference.md](api-reference.md).

A CMS plugin declares both `configureManagedCollection` (first-time setup / field
mapping) and `syncManagedCollection` (re-sync existing collection) modes in `framer.json`.

---

## ManagedCollection interface

```typescript
interface ManagedCollection {
    id: string
    name: string

    // Items
    getItemIds(): Promise<string[]>
    addItems(items: ManagedCollectionItemInput[]): Promise<void>
    // Upsert: adds new items, updates existing items matched by id.
    removeItems(ids: string[]): Promise<void>
    setItemOrder(ids: string[]): Promise<void>

    // Fields
    getFields(): Promise<ManagedCollectionField[]>
    setFields(fields: ManagedCollectionFieldInput[]): Promise<void>
    // Replaces the entire field schema. Existing data for removed fields is lost.

    // Plugin Data (per-collection)
    setPluginData(key: string, value: string | null): Promise<void>
    getPluginData(key: string): Promise<string | null>
    // 2kB per entry, 4kB per collection. Strings only.

    // Activation
    setAsActive(): Promise<void>
}
```

**Critical constraints**
- `addItems()` is an **upsert** — adds new items and updates existing ones matched by `id`. No need to check existence first.
- `ManagedCollection` has **no `getItems()`** — only `getItemIds()`. You can read item IDs, never existing field data. Design sync logic around this.
- `setFields()` replaces the **whole** schema; data for any field you drop is lost.

---

## ManagedCollectionFieldInput

```typescript
interface ManagedCollectionFieldInput {
    id: string                              // Stable identifier, used in fieldData keys
    name: string                            // Display name in Framer CMS
    type: CollectionFieldType
    userEditable?: boolean                  // Default false for managed collections.
                                            // If true, user can edit this field in CMS UI
                                            // and sync won't overwrite it.

    // Type-specific properties:
    cases?: { id: string; name: string }[]  // Required for "enum"
    collectionId?: string                   // Required for "collectionReference" / "multiCollectionReference"
    allowedFileTypes?: string[]             // Optional for "file"
    fields?: ManagedCollectionFieldInput[]  // Required for "array" (gallery sub-fields)
}
```

### CollectionFieldType — all values

```typescript
type CollectionFieldType =
    | "boolean" | "color" | "number" | "string" | "formattedText"
    | "image" | "file" | "link" | "date" | "enum"
    | "collectionReference" | "multiCollectionReference" | "array"
    | "divider" | "unsupported"
```

---

## ManagedCollectionItemInput

```typescript
interface ManagedCollectionItemInput {
    id: string                                      // Unique item identifier (used for upsert matching)
    slug: string                                    // Unique across collection, max 64 chars
    draft: boolean                                  // true = draft, false = published
    fieldData: Record<string, FieldDataEntryInput>  // Keys are field IDs
}
```

### FieldDataEntryInput — every value MUST declare its type

Never pass raw values. `fieldData: { title: "x" }` is silently ignored — it must be `{ title: { type: "string", value: "x" } }`.

```typescript
{ type: "string", value: string }
{ type: "number", value: number }
{ type: "boolean", value: boolean }
{ type: "date", value: string | null }              // ISO 8601, e.g. "2024-06-15T10:30:00Z"
{ type: "link", value: string | null }
{ type: "image", value: string | null }             // URL
{ type: "file", value: string | null }              // URL
{ type: "color", value: string | null }             // hex
{ type: "formattedText", value: string, contentType?: "html" }
{ type: "enum", value: string }                     // the case ID, not the display name
{ type: "collectionReference", value: string }      // item ID or slug
{ type: "multiCollectionReference", value: string[] }
{ type: "array", value: ArrayItemInput[] }          // gallery: nested items with own fieldData
```

```typescript
interface ArrayItemInput {
    id: string
    fieldData: Record<string, FieldDataEntryInput>
}
```

### Default values for missing source fields

| Type | Default |
|------|---------|
| `string`, `formattedText` | `""` |
| `boolean` | `false` |
| `number` | `0` |
| `image`, `file`, `link`, `date`, `color` | `null` |
| `collectionReference` | `null` |
| `multiCollectionReference` | `[]` |
| `array` | `[]` |
| `enum` | First case ID |

---

## CMS permissions

```typescript
import { framer, type ProtectedMethod } from "framer-plugin"

export const SYNC_METHODS = [
    "ManagedCollection.setFields",
    "ManagedCollection.addItems",
    "ManagedCollection.removeItems",
    "ManagedCollection.setPluginData",
] as const satisfies ProtectedMethod[]

// At sync start (imperative):
if (!framer.isAllowedTo(...SYNC_METHODS)) {
    framer.closePlugin("Insufficient permissions", { variant: "error" })
}

// In React components (reactive):
import { useIsAllowedTo } from "framer-plugin"
const canSync = useIsAllowedTo(...SYNC_METHODS)
<div role="button" aria-disabled={!canSync} onClick={canSync ? handleSync : undefined}>Sync</div>
```

`framer.isAllowedTo()` is **synchronous** — returns `boolean`, do NOT `await` it. "Get" methods (`getItemIds`, `getFields`, `getPluginData`) do not require permission checks.

---

## CMS plugin entry point (main.tsx)

Every official CMS plugin follows this shape: in `syncManagedCollection` mode, attempt a **silent sync and close** — only show UI when there's no stored config or sync fails. This is the single most important CMS pattern.

```typescript
import { framer } from "framer-plugin"
import "framer-plugin/framer.css"

const collection = await framer.getActiveManagedCollection()
const previousSourceId = await collection.getPluginData("sourceId")
const previousSlugFieldId = await collection.getPluginData("slugFieldId")

if (framer.mode === "syncManagedCollection" && previousSourceId && previousSlugFieldId) {
    if (!framer.isAllowedTo(...SYNC_METHODS)) {
        framer.closePlugin("Insufficient permissions", { variant: "error" })
    }
    await framer.hideUI()
    await syncCollection(collection, previousSourceId, previousSlugFieldId)
    framer.closePlugin("Sync complete", { variant: "success" })
}

// Fall through to configuration UI
const root = document.getElementById("root")!
createRoot(root).render(<App collection={collection} />)
```

---

## Sync algorithm (universal)

Use the "unsynced items" diff — only remove items no longer in the source. **Never** remove-all + re-add: that destroys user-editable field data and opens a race window where items briefly vanish.

```typescript
async function syncCollection(collection: ManagedCollection, items: ItemData[]) {
    const existingIds = new Set(await collection.getItemIds())
    const incomingIds = new Set(items.map(i => i.id))

    const collectionItems = items.map(item => ({
        id: item.id,
        slug: generateSlug(item.title, item.id),
        draft: false,
        fieldData: {
            title: { type: "string", value: item.title },
            // ... only sync-managed fields — omit user-editable fields
        },
    }))

    const staleIds = [...existingIds].filter(id => !incomingIds.has(id))

    await collection.setFields(allFields)          // both sync + user-editable fields
    if (staleIds.length > 0) {
        await collection.removeItems(staleIds)     // remove only stale items
    }
    await collection.addItems(collectionItems)     // upsert current items
}
```

**Order**: `setFields` → `removeItems` (stale only) → `addItems` (all current). Because `addItems` upserts, existing items get sync-managed fields updated while user-editable fields (absent from `fieldData`) are preserved.

---

## User-editable fields

Managed collections can carry fields the user edits by hand alongside synced fields. Set `userEditable: true` and **never include that field in `fieldData`** during sync — omitting a field on upsert leaves its value untouched.

```typescript
const SYNC_FIELDS: ManagedCollectionFieldInput[] = [
    { id: "title", name: "Title", type: "string" },
    { id: "thumbnail", name: "Thumbnail", type: "image" },
]
const USER_FIELDS: ManagedCollectionFieldInput[] = [
    { id: "hidden", name: "Hidden", type: "boolean", userEditable: true },
    { id: "featured", name: "Featured", type: "boolean", userEditable: true },
]

// User-editable fields first → they appear right after the Slug column in the CMS UI
const ALL_FIELDS = [...USER_FIELDS, ...SYNC_FIELDS]

function mapItems(items: SourceItem[]): ManagedCollectionItemInput[] {
    return items.map(item => ({
        id: item.id,
        slug: generateSlug(item.title, item.id),
        draft: false,
        fieldData: {
            title: { type: "string", value: item.title },
            thumbnail: { type: "image", value: item.thumbUrl },
            // DO NOT include "hidden" / "featured" — they're user-editable
        },
    }))
}
```

---

## Slug generation

Slugs must be **unique** within a collection and **max 64 chars**. When titles can collide, append a unique suffix (e.g. the source ID) and truncate the title portion to leave room.

```typescript
const SLUG_MAX = 64
const SUFFIX_LEN = 12  // "-" + 11-char ID

function generateSlug(title: string, id: string): string {
    const slug = title
        .toLowerCase()
        .normalize("NFD")
        .replace(/[̀-ͯ]/g, "")   // strip diacritics
        .replace(/[^a-z0-9]+/g, "-")
        .replace(/^-+|-+$/g, "")
    const trimmed = slug.slice(0, SLUG_MAX - SUFFIX_LEN).replace(/-$/, "")
    return `${trimmed}-${id}`
}
```

The official plugins' simpler slugify (no suffix — use only when titles are guaranteed unique):

```typescript
function slugify(value: string): string {
    return value
        .toLowerCase()
        .replace(/[^\p{Letter}\p{Number}()]+/gu, "-")
        .replace(/^-+|-+$/gu, "")
}
```

Validate uniqueness before `addItems` (Airtable plugin pattern):

```typescript
const seenSlugs = new Set<string>()
for (const item of items) {
    if (seenSlugs.has(item.slug)) throw new Error(`Duplicate slug: ${item.slug}`)
    seenSlugs.add(item.slug)
}
```

---

## Stable IDs from sources without them (RSS, scraped feeds)

```typescript
function simpleHash(input: string): string {
    let hash = 0
    for (let i = 0; i < input.length; i++) {
        hash = (hash << 5) - hash + input.charCodeAt(i)
        hash = hash & hash
    }
    return Math.abs(hash).toString(16).padStart(8, "0")
}
```

---

## Plugin-data keys

Store sync state on the **collection** (`collection.setPluginData`) so it persists across collaborators — not in `localStorage`, which is per-device.

```typescript
const PLUGIN_KEYS = {
    SOURCE_ID: "pluginSourceId",
    SLUG_FIELD_ID: "pluginSlugFieldId",
    LAST_SYNCED: "pluginLastSynced",
    TABLE_NAME: "pluginTableName",
} as const

await collection.setPluginData(PLUGIN_KEYS.SOURCE_ID, sourceId)
await collection.setPluginData(PLUGIN_KEYS.LAST_SYNCED, lastSyncedIso)  // compute the ISO string in plugin code
const previousSourceId = await collection.getPluginData(PLUGIN_KEYS.SOURCE_ID)
```

> Backend note: `new Date().toISOString()` is fine in plugin (browser) code. If a Val Town backend produces the timestamp instead, `Date.now()`/argless `new Date()` are unavailable in **Workflow scripts** there — stamp the time outside the script or in an HTTP handler.

---

## Item order preservation

```typescript
await collection.addItems(collectionItems)
await collection.setItemOrder(collectionItems.map(item => item.id))
```

---

## Cross-collection lookup

Scan all collections to find ones already linked to a given source (Notion plugin):

```typescript
async function getExistingDatabaseMap(): Promise<Map<string, string>> {
    const map = new Map<string, string>()
    for (const collection of await framer.getCollections()) {
        const dbId = await collection.getPluginData("databaseId")
        if (dbId) map.set(dbId, collection.id)
    }
    return map
}
```

---

## Field definitions — static vs dynamic

**Static** (simple sources like RSS): hard-code the field list.

```typescript
const FIELDS: ManagedCollectionFieldInput[] = [
    { type: "string", name: "Title", id: "title" },
    { type: "link", name: "Link", id: "link" },
    { type: "date", name: "Date", id: "date" },
    { type: "formattedText", name: "Content", id: "content" },
]
```

**Dynamic** (Airtable, Notion, Sheets): infer field types from the source, let the user customise the mapping in UI, then call `setFields()` with the result.

---

## Close-warning during sync

```typescript
async function syncWithWarning(collection: ManagedCollection) {
    try {
        await framer.setCloseWarning("Sync in progress. Closing will cancel the sync.")
        await performSync(collection)
        framer.closePlugin("Sync complete", { variant: "success" })
    } catch (error) {
        if (error instanceof FramerPluginClosedError) return
        throw error
    } finally {
        await framer.setCloseWarning(false)
    }
}
```

---

## Resilient, progress-tracked sync

Don't let one failed item abort the whole sync; cap concurrency.

```typescript
import pLimit from "p-limit"
const limit = pLimit(5)

const promises = items.map((item, i) =>
    limit(async () => {
        const result = await fetchItemDetails(item.id)
        onProgress({ current: i + 1, total: items.length })
        return result
    })
)

const results = await Promise.allSettled(promises)
const ok: CollectionItem[] = []
let errorCount = 0
for (const r of results) {
    if (r.status === "fulfilled" && r.value !== null) ok.push(r.value)
    else if (r.status === "rejected") errorCount++
}

await collection.addItems(ok)
if (errorCount > 0) framer.notify(`Synced with ${errorCount} errors`, { variant: "warning" })
```

---

## CMS-specific pitfalls

- **Forgetting the field-data type wrapper.** `{ title: "Hello" }` is silently ignored; it must be `{ title: { type: "string", value: "Hello" } }`. First thing to check when "sync runs but nothing appears."
- **Not attempting silent sync.** Always showing UI in `syncManagedCollection` mode is the most common CMS mistake. Try sync first, close on success.
- **remove-all + re-add.** Destroys user-editable data and creates a window where items vanish. Always diff and remove only stale IDs.
- **No `getItems()` on ManagedCollection.** You cannot read existing field values to merge — only IDs. The upsert-omit trick is the only way to preserve user data.
- **Slug collisions / >64 chars** silently fail or truncate. Validate + suffix + truncate.
- **`setPluginData` quota** is 4kB per collection total. Pack structured state into a single JSON-string key rather than many keys.

---

## Non-managed Collection (for the `collection` mode)

For collections the plugin doesn't own (user-editable):

```typescript
interface Collection {
    id: string
    name: string
    managedBy: "user" | string     // "user" for unmanaged, else the managing plugin ID
    addFields(fields: CollectionFieldInput[]): Promise<void>
    addItems(items: CollectionItemInput[]): Promise<void>
    // ...
}
```

Use `framer.getActiveCollection()` in `collection` mode. Permission strings are the `Collection.*` family (`Collection.addItems`, `Collection.addFields`, `Collection.removeItems`, `Collection.setPluginData`, `Collection.setItemOrder`, ...).
