# HLS Video Streaming (`.m3u8`) `[C]`

Chrome/Firefox do **not** natively support HLS streams. A plain `<video src="...m3u8">` will either fail or play the lowest quality rendition permanently. Safari handles HLS natively.

**Fix:** Use HLS.js via dynamic import with silent fallback:

```typescript
let HlsModule = null
let hlsImportAttempted = false

async function loadHls() {
    if (hlsImportAttempted) return HlsModule
    hlsImportAttempted = true
    try {
        const mod = await import("https://esm.sh/hls.js@1?external=react,react-dom")
        HlsModule = mod.default || mod
    } catch {
        HlsModule = null // Fallback to native video
    }
    return HlsModule
}

function attachHls(videoEl, src) {
    if (typeof window === "undefined") return null // SSR guard
    const Hls = HlsModule
    if (src.includes(".m3u8") && Hls?.isSupported()) {
        const hls = new Hls({ startLevel: -1, capLevelToPlayerSize: true })
        hls.loadSource(src)
        hls.attachMedia(videoEl)
        hls.on(Hls.Events.MANIFEST_PARSED, () => videoEl.play().catch(() => {}))
        hls.on(Hls.Events.ERROR, (_, data) => {
            if (data.fatal) {
                data.type === Hls.ErrorTypes.NETWORK_ERROR
                    ? hls.startLoad()
                    : hls.destroy()
            }
        })
        return hls
    }
    videoEl.src = src // MP4/webm or Safari native HLS
    videoEl.play().catch(() => {})
    return null
}
```

**Key points:**
- Dynamic import avoids breaking the component if CDN is unreachable
- `capLevelToPlayerSize: true` prevents loading 4K for a 400px player
- Must destroy HLS instances on cleanup to prevent memory leaks
- Use `cancelled` flag in effects to prevent stale attachment after fast navigation
- Works on Framer canvas and published site
