# Framer Labs

A collection of resources for working with Framer.

## [Skills](skills/)

Claude Code skills that teach Claude how to build Framer components and plugins. See [skills/README.md](skills/README.md) for installation instructions and detailed coverage.

| Skill | Description |
|-------|-------------|
| [framer-code-components-overrides](skills/framer-code-components-overrides/) | Code Components, Code Overrides, property controls, WebGL shaders, and common patterns |
| [framer-plugins](skills/framer-plugins/) | Framer Plugin SDK — ManagedCollection API, CMS sync, plugin modes, UI patterns, permissions, Marketplace submission |

## [Hacks](hacks/)

Workarounds and tricks for Framer.

| Hack | Description |
|------|-------------|
| [Hide 'Made in Framer' badge](hacks/Hide%20'Made%20in%20Framer'%20badge.md) | Hide the Framer badge on published sites |
| [Safari SVG animation fix](hacks/Safari%20SVG%20animation%20fix.md) | Fix choppy SVG animations on Safari |
| [Magnetic hover effect](hacks/Magnetic%20hover%20effect.md) | Elements that follow the cursor on hover |
| [Force NPM package version](hacks/Force%20NPM%20package%20version.md) | Bypass Framer's package cache with esm.sh |
| [Shared state between overrides](hacks/Shared%20state%20between%20overrides.md) | Make multiple elements react to each other |
| [Show element only once](hacks/Show%20element%20only%20once.md) | Show popups/banners only on first visit |
| [Auto-sized text fix](hacks/Auto-sized%20text%20fix.md) | Fix text collapse in auto-sized components |
| [Dynamic FAQ accordion](hacks/Dynamic%20FAQ%20accordion.md) | Auto-close other FAQ items when one opens, via a shared store |
| [Overlay deep link from URL](hacks/Overlay%20deep%20link%20from%20URL.md) | Open a Framer overlay from a URL parameter via React fiber traversal |

## Changelog

**2026-05-13** — Added Overlay deep link from URL hack (open a Framer overlay from `?member=<slug>` by calling Framer's `onTap` handler via React fiber traversal instead of dispatching synthetic events). Skill: new "Triggering Framer-Attached Handlers from Code" section with the `findFiberHandler` helper and debugging snippets. Also sync'd skill with corrections from a prior revision — replaced the old `setTimeout(100)` CMS-timing pattern with the `resolvePlainText` gating pattern, replaced the recursive `cloneElement` text-manipulation example with the `props.text`-first / children-fallback pattern, and removed the incorrect blanket `startTransition` rule.

**2026-04-17** — Added Dynamic FAQ accordion hack (shared-store override that toggles `Closed`/`Opened` variants without wrapping the list in a single component).

**2026-03-24** — Added Framer Marketplace requirements to framer-plugins skill

**2026-03-11** — Added HLS video streaming pattern (dynamic HLS.js import with silent fallback, SSR guard, adaptive bitrate). New pitfall entry for pixelated .m3u8 streams in Chrome.

**2026-03-02** — Added framer-plugins skill (Plugin SDK, ManagedCollection API, CMS sync). Restructured repo.

**2026-02-10** — Added React Portal pattern for z-index stacking context, loading states with scroll lock, and easing curves for lerp animations with initial distance tracking.

**2026-02-02** — Added advanced gotchas: startTransition requirement, ResponsiveImage/File defaultValue workaround, variant-to-font-weight mapping, and more property control tips.

**2026-01-30** — First skill + hacks

## License

MIT
