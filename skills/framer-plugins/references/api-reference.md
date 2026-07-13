# Framer Plugin API Reference

Detailed API documentation for the `@framer/plugin` SDK. For a concise overview, see [SKILL.md](SKILL.md).

> Package renamed (Framer 3.0, `@framer/plugin@4.0.1`, 2026-06-16): import from `@framer/plugin`. The old `framer-plugin` package is deprecated on npm. peerDeps: react ^18.2.0, react-dom ^18.2.0, csstype ^3.1.1.

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
// SDK 3.5.2 (Jul 2025): setMenu adds a global menu to the plugin window header.
// Returns Promise<void>. Not a protected method.

framer.showContextMenu(menuItems: MenuItem[], config: ContextMenuConfig): Promise<void>
// SDK 3.5.2 (Jul 2025): context menu anywhere in the plugin UI. Not protected.

framer.navigateTo(nodeId: string, opts?: NavigableOptions): Promise<void>
// SDK 3.7.0 (Sep 2025): selects + zooms a node into view.
// NavigableOptions = { select?: boolean (default true); zoom?: boolean }
// Not a protected method (no isAllowedTo gate needed).
// Instance forms: node.navigateTo(opts?), collectionItem.navigateTo(opts?), codeFile.navigateTo().
```

### Collection Access

```typescript
framer.getActiveManagedCollection(): Promise<ManagedCollection>
framer.getActiveCollection(): Promise<Collection>
framer.getManagedCollections(): Promise<ManagedCollection[]>   // plural — still current
framer.getCollections(): Promise<Collection[]>
framer.createManagedCollection(): Promise<ManagedCollection>
// framer.getManagedCollection() (singular) is @deprecated — use getActiveManagedCollection().
// Plugins 3.0 (Mar 2025): plugins can access ALL Collections (not just plugin-managed),
// write to unmanaged collections, and call createManagedCollection().
```

### Canvas / Node Methods

```typescript
// Verified against @framer/plugin@4.0.1:
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
framer.subscribeToOpenCodeFile(callback: (codeFile: CodeFile | null) => void): () => void
// SDK 3.7.0 (Sep 2025): fires when the user opens/switches code files. Not protected.

// Tree + geometry (verified in @framer/plugin v3+, production plugin 2026-06):
framer.getParent(nodeId): Promise<AnyNode | null>      // also node.getParent()
framer.getChildren(nodeId): Promise<CanvasNode[]>
framer.setParent(nodeId, parentId, index?): Promise<void>
// SDK v4: setParent now also accepts an UnknownNode (additive — existing calls unchanged).
framer.getRect(nodeId): Promise<Rect | null>           // also node.getRect()
framer.cloneNode(nodeId): Promise<AnyNode | null>
// SDK v4 (Jun 2026): WebPageNode.clone() and DesignPageNode.clone().
// SDK 3.8.0 (Nov 2025): plugins can read/create/edit nodes on Design Pages via the same Nodes + Selection APIs.
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

**Minor additions (v4, Jun 2026):** `framer.getLocalizationGroups()` gained an optional filter;
new frame/text link attributes `linkSmoothScroll`, `linkTrackingId`, `linkRelValues`,
`linkPreserveParams`; responsive image controls can now be set on Component instances.

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
    ...methods: [ProtectedMethod, ...ProtectedMethod[]],
    callback: (isAllowed: boolean) => void
): () => void
// METHODS FIRST, callback LAST (verified against the 4.0.1 d.ts — an earlier
// version of this file had the argument order reversed).
```

### Draggable Component

```typescript
import { Draggable } from "@framer/plugin"

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

## CMS types — see cms-managed-collections.md

The full CMS surface — `ManagedCollection`, `ManagedCollectionFieldInput`, `CollectionFieldType`, `ManagedCollectionItemInput`, `FieldDataEntryInput` (every value needs an explicit `{ type, value }` wrapper), missing-field defaults, and the non-managed `Collection` interface — lives in **[cms-managed-collections.md](cms-managed-collections.md)**, which is self-contained for CMS work. It used to be duplicated here; the CMS file is the single copy now.

---

## ProtectedMethod — verified surface (@framer/plugin@4.0.1)

`ProtectedMethod` is defined in the 4.0.1 d.ts as `AllMethods & string` (derived from the `methodToMessageTypes` table; entries with empty message arrays — the reads — are excluded from the protected union). The names below are the real ones (verified against the installed `.d.ts`), grouped by area. This is a representative subset, not the whole union; when in doubt, let TypeScript narrow it (`satisfies ProtectedMethod[]`).

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
import { useIsAllowedTo } from "@framer/plugin"

// Reactive permission check — re-renders when permissions change
const canSync: boolean = useIsAllowedTo(
    "ManagedCollection.addItems",
    "ManagedCollection.removeItems"
)
```

---

## Error Handling

```typescript
import { FramerPluginClosedError } from "@framer/plugin"

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
