# Framer Plugin Marketplace Requirements

Everything needed to ship a plugin to the Framer Marketplace. Docs claims re-verified against the live official pages 2026-07-13; the "observed" notes are first-hand submission experience (2026-06/07).

> **How review actually works (2026-07).** The official help article states plugins are "published immediately after you submit them. There is no review process or waiting period" — and publication is indeed immediate. But an **automated post-publication code scan** reads the submitted zip's actual code, typically lands **within hours**, and can flag the plugin ("Needs Changes" + email) or pull a live listing; it also **re-scans periodically after publication**. The official plugin-requirements page frames the quality bar as *recommendations*, not rejection criteria — empirically, the scan enforces the security/permission subset of them. Practical stance: **treat the recommendations as requirements and build for the scan.**

---

## Publishing Workflow

### Pack & Submit

```bash
npm run pack          # Generates plugin.zip at project root
```

1. Go to the Marketplace creator dashboard → **New Plugin** (current site naming: Marketplace → **Post > Plugin**)
2. Upload the generated `plugin.zip`
3. Fill in the listing form and submit

### Updates

Same workflow for updates:
```bash
npm run pack          # Regenerate zip with new code
```
Upload the new zip via the dashboard's **New Version** flow. Per the current docs you can upload new versions and edit plugin details at any time; expect the automated scan to re-run on new code. Metadata edits (description, images) apply instantly.

### Submission identity — manifest id and name (learned live 2026-07-08)

- **The plugin's marketplace name comes from `framer.json`'s `name`, NOT the zip filename.** The New Plugin overlay has a display glitch showing the zip's filename in its "Name" field (Framer confirmed the bug) — don't rename the zip to compensate; upload `pack`'s default `plugin.zip`.
- **"Title failed validation" means the name is already taken** by another marketplace submission (client-side check, fires before any network request).
- **`framer.json`'s `id` (the API `manifestId`) is globally unique and BURNED by every NEW submission attempt that reaches the server, success or fail** (retry = 409 unique-constraint). Each fresh submission attempt needs a new id (`openssl rand -hex 3`) + repack. **Version updates are the exception: the New Version flow reuses the live record's id — never change it for an update**, or you create a different plugin.
- **A partial/failed submission leaves an ORPHAN plugin record** (200 on the API, null `slug`/`submittedAt`, absent from the dashboard) that blocks both the name and the manifestId until an API `DELETE` or Framer support clears it; a fresh id + fresh name sidesteps it.
- The Creator Dashboard list renders server-side and can look stale; confirm real state via `GET https://api.framer.com/site/v1/plugins/<pluginId>` from a logged-in framer.com console.

### The automated scan — what it checks

Publication is immediate; the scan verdict typically lands **within hours** (two first-hand data points, 2026-06 and 2026-07), and re-scans hit already-published plugins. It pattern-matches a rubric — `isAllowedTo()` on *every* protected API call, plugin-data size limits (2kB/entry), graceful failure on denied permissions, no commercial/license identifiers in the bundle. Make the guards **explicit and greppable**; implicit/clever handling risks a flag. (An earlier version of this file described a documented ~3-week human review pipeline with a 7-day requirements check and 14-day design review — the current official docs contain no such thing; treat any multi-week expectation as obsolete.)

> **Observed 2026-06 (a *published* plugin auto-pulled by a security re-scan — the same reviewer also runs post-publication):** the email cited "malicious/unsafe code." The actual greppable findings were: (1) **commercial/license identifiers in the client bundle** (LemonSqueezy store/product/variant IDs, checkout URL, and direct `api.lemonsqueezy.com` activate/validate calls) — the bundle is extractable, so move all license logic behind a backend the client calls; (2) the user's **API key stored in shared `collection.setPluginData`** (collaborator-readable) — move secrets to `localStorage`; (3) **one-time `framer.isAllowedTo()` checks + unwrapped `setPluginData` writes** — use the reactive `useIsAllowedTo` hook to disable UI and wrap every protected write in try/catch. None of it was actually malicious — "harmful code" is the rubric's label for these patterns. Fix them proactively even if currently approved.

> **Observed 2026-07 (first-hand submission, "Needs Changes" within hours — confirms and sharpens the rubric):** the automated code review reads the ZIP's actual code and flagged protected operations "without consistently checking that the required permissions are available beforehand" even though the flows WERE gated at the UI level (reactive `useIsAllowedTo` disabling the CTA, distant early-return in the handler). What satisfies it, per protected call site: (1) an **explicit `framer.isAllowedTo()` check ADJACENT to the call** — not only a reactive hook far upstream; (2) **try/catch around the call** with clear user-facing failure copy (a silent early-return on a blocked click also reads as a failure to "handle gracefully"); (3) UI disabled when the permission is missing. **Dead/unreachable code performing ungated protected calls is review surface too** — the reviewer pattern-matches call sites, not reachability; delete parked code rather than argue it never runs. The same verdict asked to **"submit the original source files alongside the production bundle rather than only a minified build."** Treat that as a requirement, not a suggestion: `framer-plugin-tools pack` zips whatever `dist/` holds after the build script runs, so add `build.sourcemap: true` (Vite) plus a post-build step copying `src/` + configs + a build-repro README into `dist/source/`. Verify before submitting: grep the packed bundle — guards survive minification as `.isAllowedTo` property accesses; confirm zero commercial identifiers (sourcemaps and shipped sources included). A "Needs Changes" verdict is fixed via the dashboard's **New Version** flow on the same plugin record (same `framer.json` id — see "Submission identity" above), then reply to the review email so the team restores the listing.

> **Observed 2026-11 (five rejections across two paid CMS plugins, then two approvals: the credential rule widened and the scan re-reads old code):**
> 1. **A license key in project `framer.setPluginData()` is now rejected** as a credential every collaborator can read and reuse. The same code had passed earlier reviews, so "no credentials in shared plugin data" now covers license keys and activation ids, not only API keys. Fix: keep the license on your backend per Framer project ID, and keep only a non-secret project-ID marker in plugin data (pattern in [pitfalls.md](pitfalls.md#licensed-plugins--project-remix)).
> 2. **Any `getPluginData` / `setPluginData` call on a credential key is flagged, whatever it is for.** A one-time migration that read the old key, sent it to the backend and then deleted it with `setPluginData(key, null)` was rejected exactly like storing it. The reviewer names the minified function and its call sites ("search for X: activate, claim, validate") and asks to remove the reads and writes.
> 3. **Unchanged code is re-reviewed on every submission.** A migration that moved an API key out of collection plugin data into `localStorage`, untouched for three approved versions, was rejected on the fourth. A past approval is not evidence that a pattern is accepted. Grep the bundle for every plugin-data key that ever held a credential, old migrations included, before each submission.
>
> Consequence for users: anyone whose credential only exists in an old version's plugin data enters it again (keep a re-entry screen), unless your backend can find it on its own, which licenses usually allow (see pitfalls.md).
>
> **"Approved, with optional improvements"** is also a verdict: the plugin publishes and the email lists suggestions. Treat them as the next round's requirements. The ones seen so far:
> - Say in the listing what license activation and checks send, and to which host (the license key, the Framer project ID, your backend's domain). A listing edit, so it needs no new version.
> - Don't report success when a follow-up protected write failed: a sync whose metadata `setPluginData` failed must not end in a plain success message or a success `closePlugin` toast.
> - Await plugin data writes behind a settings control and roll the control back when the write fails.
> - Check the length of externally sourced strings (titles from a third-party API) before `setPluginData` (2kB per entry), and say clearly when a value can't be saved.
> - Ask for confirmation before a sync removes CMS items, with a summary of what will go.

### Submission Statuses (Creator Dashboard)

Observed first-hand: **Needs Changes** (revision requested; feedback by email) and **Published** (live on the Marketplace). **In Review** and **Rejected** are plausible but not confirmed against the current dashboard — the full four-status list is documented nowhere official.

### Contact

For questions about submission or review: **creators@framer.com**

---

## Listing Assets — Exact Specs

| Asset | Required Spec |
|-------|--------------|
| Plugin thumbnail | **1600 × 1200 px**, high-quality, clutter-free |
| Plugin icon | 30×30, **SVG or bitmap** (supply bitmaps at 90×90 px for retina), set in `framer.json` |
| Creator avatar | 200 × 200 px |
| Creator banner | 2400 × 400 px |

Thumbnails must be clutter-free and accurately represent the plugin — no misleading visuals.

---

## UI Requirements (officially "recommendations" — build to them anyway)

The current plugin-requirements page frames these as best-practice recommendations, not rejection criteria. Empirically the automated scan enforces at least part of them, and a flagged listing costs a fix-and-resubmit cycle:

| Requirement | Detail |
|-------------|--------|
| **Light + dark mode** | Plugin UI must look correct in both. Framer users switch between them. |
| **English only** | All plugin UI text must be in English. |
| **Intuitive, polished UX** | Aligned with Framer's design language. Clean and minimal. |
| **High-quality icon** | Used in `framer.json` and displayed in the Marketplace (SVG or 90×90 bitmap). |
| **No unrelated ads** | No promotional content for other products inside the plugin. |
| **Accurate description** | Plugin must solve exactly what the listing claims. No feature overpromising. |

---

## Performance Requirements

- Must load **quickly** without blocking Framer content
- Avoid **high memory or CPU consumption** — plugins run inside an iframe alongside a heavy design app
- Must be **extensively tested** to prevent crashes and bugs
- Test across **different project states and browsers**

---

## Security & Legal Requirements

| Rule | Implication |
|------|-------------|
| Use only secure, transparent external services | No hidden/obfuscated endpoints |
| Don't over-rely on third-party services for core features | Core functionality should degrade gracefully if a 3rd-party is down |
| Creators must hold IP rights for all assets | No unlicensed icons, images, or code |
| Properly credit open-source licenses | Follow OSS license terms; display attribution where required |
| No harmful, illegal, or adult content | Instant rejection |
| Write clean, documented, modular code | The automated scan reads the zip's actual code; ship original sources alongside the bundle |

---

## Pricing & Monetisation Rules

- If the plugin has **paid features**, the pricing model must be **clearly explained** in the listing
- All prices must be displayed in **USD only**
- Clearly disclose any **required authentication** (API keys, OAuth, subscription) in the listing description
- Listing fees: **none**
- **Creators keep 100% of plugin revenue**

---

## Post-Publication Obligations

Rejection can also happen to existing published plugins if these obligations aren't met:

- **Keep the plugin updated** alongside Framer platform changes (SDK updates, API changes)
- **Respond promptly** to user-reported issues
- **Set transparent support expectations** (response time, support channels)
- **Maintain accuracy** — update the listing if features change

---

## Pre-Submission Checklist

Before submitting, verify:

- [ ] Plugin icon and name are finalized and accurate
- [ ] All core functionality is tested thoroughly
- [ ] Tested across different project states and browsers
- [ ] UI looks correct in both **dark and light mode**
- [ ] All UI text is in **English**
- [ ] Thumbnail is 1600 × 1200 px and clutter-free
- [ ] Pricing model is clearly explained in the description (if paid)
- [ ] Authentication requirements are disclosed in the description
- [ ] IP rights held for all assets used
- [ ] No unrelated ads or promotional content
- [ ] All features work exactly as described
- [ ] Include test accounts or license keys with the submission (if needed for review)
- [ ] No `getPluginData` / `setPluginData` call touches a credential (API key, license key, token, activation id), old migrations included
- [ ] Paid plugins: the listing says what license activation and checks send, and to which host

---

## Private Plugins

If you need a plugin for **internal team use only** (not listed publicly), contact Framer directly to discuss private plugin options.

---

## Creator Dashboard Analytics

Once published, the dashboard provides:

| Metric | Description |
|--------|-------------|
| **Unique users** | Number of distinct creator accounts using the plugin |
| **Total uses** | Cumulative usage count across all users |

Use these to understand adoption and prioritize improvements.
