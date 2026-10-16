# Framer Labs

A collection of resources for working with Framer.

Questions, findings and help from other Framer builders: [join the Framer Labs Discord](https://discord.gg/5Kgznx7PN9).

## [Skills](skills/)

Agent skills that teach AI coding agents how to build Framer sites, components and plugins. See [skills/README.md](skills/README.md) for installation instructions and detailed coverage.

| Skill | Description |
|-------|-------------|
| [framer-code-components-overrides](skills/framer-code-components-overrides/) | Code Components, Code Overrides, property controls, WebGL shaders, and common patterns |
| [framer-agent-playbook](skills/framer-agent-playbook/) | Building and editing Framer sites through Framer Agent — what can't be undone, DSL and API traps, breakpoints, CMS, forms, and working safely in live client sites |
| [framer-plugins](skills/framer-plugins/) | Framer Plugin SDK — plugin modes, canvas insertion + code files, CMS/ManagedCollection sync, permissions, licensing, Marketplace submission |

### Contributing

The skills are community-maintained and grow from real builds: every finding comes from something that broke or cost time on an actual Framer project. Your agent can add to them. Put one line per skill in your project's `CLAUDE.md` or `AGENTS.md`:

```
framer-plugins: auto-update, contribute
framer-code-components-overrides: auto-update
```

- `auto-update` replaces your installed copy when a newer version is published here, in its own commit.
- `contribute` lets your agent offer, at the end of a session, to open a pull request with what it learned. It strips everything that comes from your project first (company and client names, domains, project ids, file paths, keys) and shows you the diff before anything is opened.

Without a line, a skill never edits itself and never opens a pull request. It only tells you when a newer version is out.

## [Hacks](hacks/)

Workarounds and tricks for Framer.

| Hack | Description |
|------|-------------|
| [Safari SVG animation fix](hacks/Safari%20SVG%20animation%20fix.md) | Fix choppy SVG animations on Safari |
| [Magnetic hover effect](hacks/Magnetic%20hover%20effect.md) | Elements that follow the cursor on hover |
| [Force NPM package version](hacks/Force%20NPM%20package%20version.md) | Bypass Framer's package cache with esm.sh |
| [Shared state between overrides](hacks/Shared%20state%20between%20overrides.md) | Make multiple elements react to each other |
| [Show element only once](hacks/Show%20element%20only%20once.md) | Show popups/banners only on first visit |
| [Auto-sized text fix](hacks/Auto-sized%20text%20fix.md) | Fix text collapse in auto-sized components |
| [Dynamic FAQ accordion](hacks/Dynamic%20FAQ%20accordion.md) | Auto-close other FAQ items when one opens, via a shared store |
| [Overlay deep link from URL](hacks/Overlay%20deep%20link%20from%20URL.md) | Open a Framer overlay from a URL parameter via React fiber traversal |

## Changelog

**2026-10-16**
- framer-code-components-overrides skill: a code component's Collection List slot arrives as an array, how to read CMS fields the item template does not show, and why CMS dates should be read in UTC.
- framer-agent-playbook skill: binding a CMS date field to text, and the date styles that do not apply.
- framer-agent-playbook skill: auto-fit text that Safari clips, and overlays that leave keyboard focus behind.

**2026-10-15**
- framer-agent-playbook skill: an SEO and speed pass. A new reference covers structured data from CMS fields, meta description fields, redirects on live sites, font files, the largest text on the first screen, analytics that miss page changes, and how to measure speed when asked. Everything in it is a suggestion to the user, never a blocker.

**2026-10-14**
- framer-agent-playbook skill: why a page transition makes a moving background in the layout template flicker and stall, and how to keep it smooth; a font change that reports success and does not apply; and the order to create pages in when their menus link to each other.

**2026-10-13**
- framer-agent-playbook skill: keep one layer per piece of content across breakpoints so the site owner edits each text once, with the tested command for reordering a layer on one breakpoint and the one layout where a copy is still needed; and push a card footer down with space-between instead of a spacer layer.
- framer-agent-playbook skill: page transitions animate the whole page, layout template included, and how to keep a shared nav and footer still while the content moves; plus how to switch a template component's variant from a page setting, and the fixed height a layout template needs.

**2026-10-12**
- framer-agent-playbook skill: what the custom code API can't do (name a snippet, add a second one, or see the snippets you add in the editor), and which Run setting keeps a pasted analytics snippet from starting twice.
- framer-code-components-overrides skill: a new first section on when code isn't needed at all (accordions, tabs and sliders are better built natively), plus notes on multi-file components, published sizes an override can't see, and why scripted navigation to another page fails on a published site.
- framer-agent-playbook skill: tested notes on Framer's auto-branching, session ids reused across projects, a native accordion built end to end through the agent, icon variables, and more ways a screenshot or a read can mislead you.
- All three skills: a pull request your agent sends back here now describes what it adds for users, never which project it came from.
- All three skills now check this repo for a newer version at the start of a session and work from the latest copy if yours is out of date. Add one line per skill to your project's CLAUDE.md or AGENTS.md to have the installed copy replaced automatically.
- The skills can now send what you learn on a real build back here as a pull request, if you opt in. Nothing goes out without your yes, and nothing from your project goes with it.

**2026-10-10**
- New framer-agent-playbook skill: what an AI agent should know before building or editing a Framer site through Framer Agent, from the deletes that can't be undone to calls that report success and do nothing, layouts that break on tablet and phone, and safe habits for live client sites; checked against Framer's own agent docs so it only covers what they leave out.
- framer-code-components-overrides skill: new notes on overrides that stop working after a publish, native Framer forms, showing new CMS items without a reload, and why CMS data is never private.
- framer-plugins skill: fixed a broken link.

**2026-07-13** — framer-plugins skill: big update for the renamed v4 plugin SDK, first-hand guidance on the Marketplace's automated code review and how to pass it (permission checks, shipping your source files), submission gotchas, and licensing advice for paid plugins; fact-checked against the live Framer docs.

**2026-06-15** — framer-plugins skill: new coverage of inserting code files onto the canvas, handling licensed plugins across project remixes, and a dedicated CMS reference; verified against the latest plugin SDK.

**2026-06-05** — Skill: documented a subtle gotcha where factoring several similar overrides through a shared generator silently drops them from Framer's override picker, and how to keep them listed.

**2026-06-04** — Skill: new sections on URL-driven filtering and styling range sliders, plus a tidy-up of the video streaming notes.

**2026-05-25** — Skill refactor with new sections on CMS in code components, color tokens, live-read refs, SSR-safe state init, concurrent rendering, and gated debug logging. Reorganised into tagged sections with the pitfalls table moved to the top.

**2026-05-13** — Added Overlay deep link from URL hack (open a Framer overlay from a URL param via React fiber traversal). Skill: new section on triggering Framer-attached handlers from code, plus a few corrections to the text and CMS-timing guidance.

**2026-04-17** — Added Dynamic FAQ accordion hack (shared-store override that toggles `Closed`/`Opened` variants without wrapping the list in a single component).

**2026-03-24** — Added Framer Marketplace requirements to framer-plugins skill

**2026-03-11** — Added HLS video streaming pattern (dynamic HLS.js import with silent fallback, SSR guard, adaptive bitrate). New pitfall entry for pixelated .m3u8 streams in Chrome.

**2026-03-02** — Added framer-plugins skill (Plugin SDK, ManagedCollection API, CMS sync). Restructured repo.

**2026-02-10** — Added React Portal pattern for z-index stacking context, loading states with scroll lock, and easing curves for lerp animations with initial distance tracking.

**2026-02-02** — Added advanced gotchas: startTransition requirement, ResponsiveImage/File defaultValue workaround, variant-to-font-weight mapping, and more property control tips.

**2026-01-30** — First skill + hacks

## License

MIT
