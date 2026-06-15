# Framer Plugin Patterns

Cross-cutting patterns drawn from the official example plugins in [framer/plugins](https://github.com/framer/plugins). For API details see [api-reference.md](api-reference.md); for CMS-only patterns (sync algorithm, slugs, user-editable fields, entry point) see [cms-managed-collections.md](cms-managed-collections.md).


## Permission Check Pattern

```typescript
import { framer, type ProtectedMethod } from "framer-plugin"

export const METHODS = [
    "ManagedCollection.setFields",
    "ManagedCollection.addItems",
    "ManagedCollection.removeItems",
    "ManagedCollection.setPluginData",
] as const satisfies ProtectedMethod[]

// Imperative (at the start of a protected operation):
if (!framer.isAllowedTo(...METHODS)) {
    framer.closePlugin("Insufficient permissions", { variant: "error" })
}

// Reactive (in React components):
import { useIsAllowedTo } from "framer-plugin"
const canRun = useIsAllowedTo(...METHODS)
<div role="button" aria-disabled={!canRun} onClick={canRun ? handleRun : undefined}>Run</div>
```

`isAllowedTo` is synchronous — never `await` it. Use `<div role="button">` rather than `<button>` (Framer overrides button styling).

---

## UI Sizing Per Screen State

Official plugins change window size based on the current screen:

```typescript
// Separate functions for each UI state
async function showLoginUI() {
    await framer.showUI({ width: 320, height: 340, resizable: false })
}

async function showConfigUI() {
    await framer.showUI({ width: 360, height: 425, minWidth: 360, minHeight: 425, resizable: true })
}

// Or in useLayoutEffect based on state:
useLayoutEffect(() => {
    framer.showUI({
        width: hasConfig ? 360 : 260,
        height: hasConfig ? 425 : 340,
        resizable: hasConfig,
    })
}, [hasConfig])
```

---

## OAuth Authentication Pattern

Used by Airtable, Google Sheets, Notion, HubSpot:

```typescript
class Auth {
    private TOKENS_KEY = "pluginTokens"

    async authorize(): Promise<Tokens> {
        // 1. Get auth URL from external worker
        const { authUrl, readKey } = await fetch("https://oauth.fetch.tools/my-plugin/authorize").then(r => r.json())
        // 2. Open browser for user to authorize
        window.open(authUrl)
        // 3. Poll for tokens
        return this.pollForTokens(readKey)
    }

    getTokens(): Tokens | null {
        const stored = localStorage.getItem(this.TOKENS_KEY)
        return stored ? JSON.parse(stored) : null
    }

    saveTokens(tokens: Tokens) {
        localStorage.setItem(this.TOKENS_KEY, JSON.stringify(tokens))
    }

    logout() {
        localStorage.removeItem(this.TOKENS_KEY)
        window.location.reload()
    }
}

export const auth = new Auth()
```

Tokens always go in `localStorage` (per-user, not shared).

---

## API Key Authentication Pattern

Simpler alternative (not explicitly in official examples, but used by community plugins):

```typescript
// Validate before persisting
async function validateApiKey(key: string): Promise<boolean> {
    try {
        const res = await fetch(`https://api.example.com/test?key=${key}`)
        return res.ok
    } catch {
        return false
    }
}

// Store in localStorage (per-user, sandboxed)
function saveCredentials(apiKey: string, sourceId: string) {
    localStorage.setItem("apiKey", apiKey)
    localStorage.setItem("sourceId", sourceId)
}
```

---

## Close Warning During Sync

Prevent accidental close during long operations (from Notion plugin):

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

## Progress Tracking

For operations with many API calls (from Notion plugin):

```typescript
interface SyncProgress {
    current: number
    total: number
    hasFinishedLoading: boolean
}

// With concurrency limiting:
import pLimit from "p-limit"
const limit = pLimit(5)

const promises = items.map((item, i) =>
    limit(async () => {
        const result = await fetchItemDetails(item.id)
        onProgress({ current: i + 1, total: items.length, hasFinishedLoading: false })
        return result
    })
)
const results = await Promise.allSettled(promises)
```

---

## Error Resilience with Promise.allSettled

Don't let one failed item abort the entire sync:

```typescript
const results = await Promise.allSettled(promises)
const items: CollectionItem[] = []
const errors: SyncError[] = []

results.forEach((result, index) => {
    if (result.status === "fulfilled" && result.value !== null) {
        items.push(result.value)
    } else if (result.status === "rejected") {
        errors.push({ index, error: result.reason })
    }
})

// Sync whatever succeeded, report errors
await collection.addItems(items)
if (errors.length > 0) {
    framer.notify(`Synced with ${errors.length} errors`, { variant: "warning" })
}
```

---

## Menu Integration

Add context menu items to the plugin:

```typescript
framer.setMenu([
    {
        label: `View ${tableName} in Airtable`,
        visible: Boolean(dataSource),
        onAction: () => window.open(`https://airtable.com/${baseId}/${tableId}`),
    },
    { type: "separator" },
    { label: "Log Out", onAction: () => auth.logout() },
])
```

---

## Canvas Mode Patterns

For plugins that insert assets onto the canvas:

```typescript
// Mode-dependent behavior (framer-plugin@3.10.3 signatures)
if (framer.mode === "canvas") {
    await framer.addImage({ image: url, altText })   // NamedImageAssetInput | File
} else {
    // "image" or "editImage" mode — set on selection
    await framer.setImage({ image: url, altText })
    framer.closePlugin()
}

// Drag-and-drop support
import { Draggable } from "framer-plugin"
<Draggable data={{ type: "image", image: fullUrl, previewImage: thumbUrl, name, altText }}>
    <div className="image-card">...</div>
</Draggable>
```

For inserting code-file / module components onto the canvas (and the selection-aware replace-in-place geometry pattern), see [api-reference.md](api-reference.md) and the Code Files section of [SKILL.md](../SKILL.md).
