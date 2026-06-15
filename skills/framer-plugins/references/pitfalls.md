# Framer Plugin Pitfalls & Workarounds

Known gotchas discovered from official examples, Framer docs, and community experience.

## UI & Styling

### Framer overrides `<button>` styles
Framer's built-in CSS has high specificity on `<button>` elements. Custom backgrounds and text colors get silently overridden, especially in light mode.

**Workarounds:**
- Use `<div role="button" tabIndex={0}>` for custom-styled interactive elements
- If you must use `<button>`, apply `!important` on `background` and `color`
- Use high-specificity selectors: `.parent .child.active { background: red !important; }`

### framer.css conflicts
Importing `"framer-plugin/framer.css"` provides Framer's base styles (fonts, colors, input styling), but can conflict with custom CSS. The import is optional but recommended for a native look.

### showUI flicker
Calling `framer.showUI()` in a `useEffect` causes a visible resize flicker. Always use `useLayoutEffect`:

```typescript
useLayoutEffect(() => {
    framer.showUI({ width: 320, height: hasConfig ? 380 : 340 })
}, [hasConfig])
```

### SVG icon centering
Icons in `framer.json` must use a 30×30 viewBox. Paths drawn at absolute coordinates can end up outside the visible area. Use `<g transform="translate(15 15)">` with centered path coordinates.

### `display:none` retriggers CSS `@keyframes` animations
When a hidden element (e.g., a tab toggled via `display:none`) becomes visible again, browsers replay all CSS `@keyframes` animations on it. This causes completion icons, fade-ins, and other one-shot animations to replay every time the user switches tabs.

**Fix**: For one-shot animations on elements inside tabs that toggle via `display:none`, either:
1. Use React state + `useRef` to apply the `animate` class only once, then remove it after the animation completes
2. For static informational states (like a "connected" confirmation), skip animations entirely — use a dedicated static class with no `@keyframes`

Option 2 is simpler and more reliable. Only use animated classes on transient states (like sync completion) where the animation is triggered by a specific event.

---

## SDK Behavior

### "Invoking protected message type" toast warnings
`setPluginData()`, `addManagedCollectionItems()`, `setManagedCollectionFields()`, and other SDK methods trigger visible toast notifications saying "Invoking protected message type ... without checking permissions first."

**What it is**: NOT a bug — it's the Framer permission system telling you to check permissions before calling protected methods. The toast appears when you invoke a protected method without first calling `framer.isAllowedTo()`.

**Fix**: Always call `framer.isAllowedTo()` before any protected method. This is a synchronous call (returns `boolean`, not a Promise):

```typescript
// Check before writing to collection pluginData
if (framer.isAllowedTo("ManagedCollection.setPluginData")) {
    await collection.setPluginData("my-key", "my-value")
}

// Check before sync operations
const canSync = framer.isAllowedTo(
    "ManagedCollection.setFields",
    "ManagedCollection.addItems",
    "ManagedCollection.removeItems",
    "ManagedCollection.setPluginData"
)
```

**Important**: `framer.isAllowedTo()` returns a plain `boolean`, not a `Promise`. Do NOT `await` it.

Source: Framer team confirmed this in community forums — see https://www.framer.com/developers/plugins-permissions

### closePlugin throws internally
`framer.closePlugin()` returns `never` and throws `FramerPluginClosedError`. Any code after it won't execute. The SDK auto-suppresses unhandled rejections of this class, but if you have a `catch` block:

```typescript
try {
    await syncCollection(collection)
    framer.closePlugin("Done", { variant: "success" })
} catch (error) {
    if (error instanceof FramerPluginClosedError) return  // Must ignore
    // Handle real errors
}
```

### addItems is upsert
`collection.addItems()` adds new items AND updates existing ones matched by `id`. You don't need to check whether an item already exists before adding it.

### Slug uniqueness is required
Items in a collection must have unique slugs. If two items share a slug, the sync may fail or produce unexpected results. Always validate before calling `addItems`.

### Slug length limit is 64 characters
Framer CMS collection item slugs have a maximum length of 64 characters. Longer slugs will be silently truncated or cause errors. When generating slugs from titles, always truncate the title portion to leave room for a unique suffix (e.g., an ID) to ensure both uniqueness and the 64-char limit.

### ManagedCollection has no getItems() method
Unlike the regular `Collection` class, `ManagedCollection` only provides `getItemIds()` — you can **only read item IDs, not field data**. This means you cannot read existing field values to compare or merge during sync. Design your sync logic around this constraint.

---

## Data Storage

### pluginData size limits
- `framer.setPluginData()`: 2kB per entry, 4kB total across all keys (project-level)
- `collection.setPluginData()`: 2kB per entry, 4kB per collection
- Values must be strings. Pass `null` to delete a key.

### localStorage is sandboxed
`localStorage` is sandboxed per-plugin origin and per-user. It's safe for API keys and tokens. It's synchronous (no async needed) and has no Framer-imposed size limits.

### When to use which storage

| Data | Storage | Reason |
|------|---------|--------|
| API keys, auth tokens | `localStorage` | Per-user, not shared, no size warnings |
| User preferences | `localStorage` | Per-user, synchronous |
| Data source ID, slug field | `collection.setPluginData()` | Shared across collaborators |
| Last sync timestamp | `collection.setPluginData()` | Shared state for the collection |
| Project-level config | `framer.setPluginData()` | Shared, but tiny (4kB total) |

Per Framer's own docs: `pluginData` should NOT store sensitive data like API keys.

### Licensed plugins & project remix

When a Framer project is remixed, both `framer.setPluginData()` and `collection.setPluginData()` values carry over to the new project unchanged. Any license keys, API credentials, or project-bound state will appear valid in the remix.

**Detection**: `framer.getProjectInfo()` returns `{ id: string, name: string }` where `id` is a hashed project ID unique to each project. Store this at activation time and compare on every load — a mismatch means the project was remixed.

**Pattern for licensed plugins:**
```typescript
const { id: currentProjectId } = await framer.getProjectInfo()

if (storedProjectId !== currentProjectId) {
    // Remixed project — clear all stale credentials so new owner goes through fresh setup
    if (framer.isAllowedTo("setPluginData")) {
        await framer.setPluginData(PD_LICENSE_KEY, null)
        await framer.setPluginData(PD_LICENSE_INSTANCE, null)
        await framer.setPluginData(PD_LICENSE_PROJECT, null)
        if (collection) {
            for (const key of [PD_API_KEY, PD_CHANNEL_ID, ...]) {
                await collection.setPluginData(key, null)
            }
        }
    }
    setLicenseActive(false)
    setScreen("splash")
    return
}
```

Note: silently skip the clear if `isAllowedTo` is false — `licenseActive = false` is correct regardless, and the clear will succeed on the next load. If your license provider supports a per-activation label (e.g. LemonSqueezy's `instance_name`), pass the project ID into it for per-project dashboard visibility. Whichever provider you use (Polar, LemonSqueezy, a Val Town backend keyed on the Framer user ID, ...), wrap validation server-side — the plugin should never call the payment provider directly.

---

## Framer Environment

### Cmd+Z / Ctrl+Z captures globally
Framer captures undo globally. Users may accidentally undo the plugin instantiation when they expect to undo text input within the plugin. This is NOT interceptable from within the plugin iframe.

### Console logs
`console.log` output appears in the browser devtools, not the terminal. To see logs:
1. Right-click the plugin iframe in Framer
2. "Inspect Element"
3. Switch to Console tab

The `npm run dev` terminal only shows Vite build/HMR output.

### Plugins run in an iframe
All plugins execute inside a sandboxed iframe. This means:
- CORS restrictions apply to external API calls
- `window.parent` access is limited
- Storage is sandboxed per-origin

### Permission errors
When a plugin attempts a protected operation without permission, the SDK throws `FramerPluginError` and shows a toast: "Insufficient permissions." Always check with `framer.isAllowedTo()` before attempting protected operations.

---

## Common Mistakes

### Not attempting silent sync
The most common mistake in CMS plugins: always showing UI in `syncManagedCollection` mode. The correct pattern is to try syncing silently first and close the plugin on success. Only fall through to UI if sync fails or no config exists.

### Forgetting field data type wrapper
Field values must include explicit type:
```typescript
// WRONG
fieldData: { title: "Hello" }

// CORRECT
fieldData: { title: { type: "string", value: "Hello" } }
```

### Using setPluginData without permission checks
`setPluginData()` and other protected methods require a `framer.isAllowedTo()` check first. Without it, you get toast warnings. With the check, `pluginData` works cleanly for shared settings. Use `localStorage` only when you need per-user, non-shared data.

### Not handling FramerPluginClosedError
If you have try/catch around async operations that end with `closePlugin()`, you must explicitly ignore `FramerPluginClosedError` or your error handling will catch it.

### Hardcoding UI dimensions
Plugin UI should resize based on the current screen state. Use `useLayoutEffect` to call `showUI` with appropriate dimensions when the state changes.

### Not checking permissions before sync
Always call `framer.isAllowedTo()` at the start of sync operations. Without it, users see unhelpful "Insufficient permissions" toasts mid-sync.

---

## Debugging Tips

- **Toast debugging**: If you see "Invoking protected message type" toasts, you're missing a `framer.isAllowedTo()` check before the protected method call. Add the check and the toast goes away.
- **Invisible button text**: If button text disappears in light mode, Framer is overriding your color. Add `color: #fff !important` or use `<div role="button">`.
- **Plugin won't load**: Check `framer.json` for valid mode names and icon paths. Icons must exist in `public/`.
- **Sync runs but nothing appears**: Check that `fieldData` values have the `{ type, value }` wrapper. Raw values are silently ignored.
- **Items disappear after sync**: If you call `removeItems` with all IDs before `addItems`, there's a race condition window. Use the unsynced-items pattern instead.
- **"setPluginData" quota exceeded**: You hit the 4kB limit. Use fewer keys or store structured data as a single JSON string.
