# Framer Plugin API Reference

Detailed API documentation for the `framer-plugin` SDK. For a concise overview, see [SKILL.md](SKILL.md).

## framer Global Object

### Mode & Identity

```typescript
framer.mode: string                    // Current plugin mode
framer.getProjectInfo(): Promise<{ id: string; name: string }>
```

### UI Management

```typescript
framer.showUI(options?: {
    position?: "center" | "top left" | "bottom left" | "top right" | "bottom right"
    width?: number
    height?: number
    minWidth?: number
    minHeight?: number
    maxWidth?: number
    maxHeight?: number
    resizable?: boolean | "width" | "height"
}): Promise<void>

framer.hideUI(): Promise<void>

framer.closePlugin(message?: string, options?: {
    variant?: "success" | "info" | "error"
}): never
// Throws FramerPluginClosedError internally. Code after this never executes.

framer.notify(message: string, options?: {
    variant?: "info" | "success" | "warning" | "error"
    durationMs?: number | typeof Infinity
    button?: { text: string; onClick: () => void }
}): void

framer.setCloseWarning(message: string | false): Promise<void>
// Show confirmation dialog when user tries to close plugin during operations.

framer.setBackgroundMessage(message: string): void
// Status text shown while plugin UI is hidden.

framer.setMenu(items: Array<
    | { label: string; onAction: () => void; visible?: boolean }
    | { type: "separator" }
>): void
```

### Collection Access

```typescript
framer.getActiveManagedCollection(): Promise<ManagedCollection>
framer.getActiveCollection(): Promise<Collection>
framer.getManagedCollections(): Promise<ManagedCollection[]>
framer.getCollections(): Promise<Collection[]>
framer.createManagedCollection(): Promise<ManagedCollection>
```

### Canvas / Node Methods

```typescript
// Verified against framer-plugin@3.10.3:
framer.addImage(image: NamedImageAssetInput | File): Promise<void>
framer.setImage(image: NamedImageAssetInput | File): Promise<void>   // sets on current selection
framer.addImages(images: readonly NamedImageAssetInput[]): Promise<void>
framer.uploadImage(image: NamedImageAssetInput | File): Promise<ImageAsset>  // upload without assigning
// NamedImageAssetInput = { image: AssetInput; altText?: string; ... } — `image` is a URL/asset, not a plain string.

framer.getImage(): Promise<{ image: string; altText?: string } | null>

framer.addText(text: string, options?: AddTextOptions): Promise<void>
framer.addSVG(svg: SVGData): Promise<void>                            // max 10kB

framer.addComponentInstance(options: {
    url: string                                          // insertURL or any module URL
    attributes?: Partial<EditableComponentInstanceNodeAttributes>
    parentId?: string
}): Promise<ComponentInstanceNode>                       // resolves to the created node — keep the id

framer.addDetachedComponentLayers(options: {
    url: string
    layout: ...
    attributes?: ...
}): Promise<FrameNode>                                   // inserts the component's layers inlined, not as an instance

framer.getSelection(): Promise<Node[]>
framer.subscribeToSelection(callback: (selection: Node[]) => void): () => void
framer.subscribeToCanvasRoot(callback: (root: Node) => void): () => void

// Tree + geometry (verified in framer-plugin v3, unicorn-studio plugin 2026-06):
framer.getParent(nodeId): Promise<AnyNode | null>      // also node.getParent()
framer.getChildren(nodeId): Promise<CanvasNode[]>
framer.setParent(nodeId, parentId, index?): Promise<void>
framer.getRect(nodeId): Promise<Rect | null>           // also node.getRect()
framer.cloneNode(nodeId): Promise<AnyNode | null>
```

`addComponentInstance`'s `attributes` accepts the full editable node attribute set
(position + pins + size), so an instance can be placed/sized at insert time.

**Node positional attributes** (readable on the node, writable via `setAttributes`):
`position`, pins `top/right/bottom/left` (px), `centerX/centerY` (%), `width/height`
(any length: px, %, fr), plus min/max constraints and `aspectRatio`.

**Replace-an-instance-in-place pattern** (swap component, keep geometry): read old
node's parent + sibling index (`getParent` + `getChildren().findIndex`), insert the
new instance, `setParent(newId, parentId, oldIndex)`, then `setAttributes` copying
`position`/pins/`centerX/Y`/`width`/`height` VERBATIM from the old node, then remove
the old one. Copy attributes, not `getRect()` pixels: verbatim copy preserves
percentage/fill sizing and stays correct inside stacks (pins are inert there — the
sibling index IS the position).

### Project-Level Plugin Data

```typescript
framer.setPluginData(key: string, value: string | null): Promise<void>
// Pass null to delete. 2kB per entry, 4kB total across all keys.

framer.getPluginData(key: string): Promise<string | null>
```

### Permissions

```typescript
framer.isAllowedTo(...methods: ProtectedMethod[]): boolean
// Synchronous check. Returns true if ALL methods are allowed.

framer.subscribeToIsAllowedTo(
    callback: (isAllowed: boolean) => void,
    ...methods: ProtectedMethod[]
): () => void
```

### Draggable Component

```typescript
import { Draggable } from "framer-plugin"

<Draggable data={{
    type: "image",
    image: "https://...",
    previewImage: "https://...",
    name: "Photo",
    altText: "Description",
}}>
    <div>Drag me</div>
</Draggable>
```

---

## ManagedCollection

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
    setAsActive(): Promise<void>  // SDK 2.2.0+
}
```

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

### CollectionFieldType — All Values

```typescript
type CollectionFieldType =
    | "boolean"
    | "color"
    | "number"
    | "string"
    | "formattedText"
    | "image"
    | "file"
    | "link"
    | "date"
    | "enum"
    | "collectionReference"
    | "multiCollectionReference"
    | "array"
    | "divider"
    | "unsupported"
```

---

## ManagedCollectionItemInput

```typescript
interface ManagedCollectionItemInput {
    id: string                                      // Unique item identifier (used for upsert matching)
    slug: string                                    // URL slug, must be unique across collection
    draft: boolean                                  // true = draft, false = published
    fieldData: Record<string, FieldDataEntryInput>  // Keys are field IDs
}
```

---

## FieldDataEntryInput — All Type Variants

Every field value must explicitly declare its type. Never pass raw values.

```typescript
// Primitives
{ type: "string", value: string }
{ type: "number", value: number }
{ type: "boolean", value: boolean }

// Date — ISO 8601 string
{ type: "date", value: string | null }
// Example: "2024-06-15T10:30:00Z"

// Link — full URL
{ type: "link", value: string | null }

// Image — URL string or null
{ type: "image", value: string | null }

// File — URL string or null
{ type: "file", value: string | null }

// Color — hex string or null
{ type: "color", value: string | null }

// Formatted Text — HTML content
{ type: "formattedText", value: string, contentType?: "html" }

// Enum — value is the case ID (not display name)
{ type: "enum", value: string }

// Collection Reference — value is an item ID or slug from the referenced collection
{ type: "collectionReference", value: string }

// Multi Collection Reference — array of item IDs/slugs
{ type: "multiCollectionReference", value: string[] }

// Array (Gallery) — nested items with their own fieldData
{ type: "array", value: ArrayItemInput[] }
```

### ArrayItemInput (for gallery/array fields)

```typescript
interface ArrayItemInput {
    id: string
    fieldData: Record<string, FieldDataEntryInput>
}
```

### Default Values for Missing Fields

When a source item has no data for a field, use type-appropriate defaults:

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

## Collection (Non-Managed)

For collections not owned by the plugin (user-editable collections):

```typescript
interface Collection {
    id: string
    name: string
    managedBy: "user" | string     // "user" for unmanaged, plugin ID for managed
    addFields(fields: CollectionFieldInput[]): Promise<void>
    addItems(items: CollectionItemInput[]): Promise<void>
    // ... other methods
}
```

Use `framer.getActiveCollection()` for the `collection` mode.

---

## ProtectedMethod — verified surface (framer-plugin@3.10.3)

`ProtectedMethod` is `keyof methodToMessageTypes` — a large union. The names below are the real ones (verified against the installed `.d.ts`), grouped by area. This is a representative subset, not the whole union; when in doubt, let TypeScript narrow it (`satisfies ProtectedMethod[]`).

```typescript
type ProtectedMethod =
    // Canvas insertion (bare names — these ARE valid protected strings)
    | "addImage" | "addImages" | "setImage" | "addSVG" | "addText"
    | "addComponentInstance" | "addDetachedComponentLayers"
    | "uploadImage" | "uploadImages" | "uploadFile" | "uploadFiles"
    // Nodes
    | "Node.setAttributes" | "Node.remove" | "Node.clone" | "Node.setPluginData"
    // ManagedCollection
    | "ManagedCollection.setFields" | "ManagedCollection.addItems"
    | "ManagedCollection.removeItems" | "ManagedCollection.setItemOrder"
    | "ManagedCollection.setPluginData" | "ManagedCollection.setAsActive"
    // Collection (non-managed) + items + fields
    | "Collection.addItems" | "Collection.removeItems" | "Collection.addFields"
    | "Collection.removeFields" | "Collection.setItemOrder" | "Collection.setFieldOrder"
    | "Collection.setPluginData" | "Collection.setAsActive"
    | "CollectionItem.setAttributes" | "CollectionItem.remove" | "CollectionItem.setPluginData"
    | "Field.remove" | "Field.setAttributes" | "EnumField.addCase" | "EnumField.setCaseOrder"
    // Top-level creators / styles
    | "createCollection" | "createManagedCollection"
    | "createColorStyle" | "createTextStyle"
    // ...and more (TextNode.*, ColorStyle.*, TextStyle.*, WebPageNode.*, EnumCase.*)
```

**"Get" methods are unprotected** and never need a check: `getItemIds`, `getFields`, `getPluginData`, `getSelection`, `getParent`, `getChildren`, `getRect`, `getProjectInfo`, `getCurrentUser`, `getCodeFiles`, `showUI`, `hideUI`, `notify`, etc.

> Note the asymmetry: the canvas `add*`/`upload*` methods use **bare** names (`"addImage"`), while collection/node methods use **namespaced** names (`"ManagedCollection.addItems"`, `"Node.setAttributes"`). Match exactly — `"Collection.addImage"` is not a thing.

---

## React Hooks

```typescript
import { useIsAllowedTo } from "framer-plugin"

// Reactive permission check — re-renders when permissions change
const canSync: boolean = useIsAllowedTo(
    "ManagedCollection.addItems",
    "ManagedCollection.removeItems"
)
```

---

## Error Handling

```typescript
import { FramerPluginClosedError } from "framer-plugin"

try {
    await syncCollection(collection)
    framer.closePlugin("Sync complete", { variant: "success" })
} catch (error) {
    if (error instanceof FramerPluginClosedError) return  // User closed plugin, ignore
    framer.notify(error.message, { variant: "error" })
}
```

The SDK auto-suppresses unhandled `FramerPluginClosedError` rejections. If you have a catch block that handles errors generically, explicitly check for and ignore this class.

---

## Official Documentation URLs

- [Reference](https://www.framer.com/developers/reference)
- [CMS / Managed Collections](https://www.framer.com/developers/cms)
- [Plugin Modes](https://www.framer.com/developers/modes)
- [Interface / showUI](https://www.framer.com/developers/interface)
- [Permissions](https://www.framer.com/developers/plugins-permissions)
- [Data Storage](https://www.framer.com/developers/storing-data)
- [Nodes](https://www.framer.com/developers/nodes)
- [Configuration / framer.json](https://www.framer.com/developers/configuration)
- [Publishing](https://www.framer.com/developers/publishing)
- [Quick Start](https://www.framer.com/developers/plugins-quick-start)
- [Changelog](https://www.framer.com/developers/changelog)
