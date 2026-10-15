# Search and speed

What makes a Framer site easier to find and faster to load, as far as it is particular to Framer.
General SEO (keywords, content plans, links from other sites) is out of scope here.

**These are defaults, never blockers.** Apply them as you build. Mention one to the user only when
it applies to what they asked for, once, in a sentence, and never repeat it in the same session.
Nothing here needs the user's sign-off before you carry on with their task, and nothing here
changes their custom code, analytics or site settings without their yes. When the user's
instructions differ, follow the user.

## CMS and pages

- **Structured data from CMS variables.** When you build a collection whose pages should rank on
  their own (a blog, docs, case studies), tell the user that the detail page's custom code can hold
  JSON-LD filled from CMS fields, so one snippet covers every item. Skip it for content that is not
  a page of its own, such as a FAQ list inside a long page. Offer to write the snippet; the user
  pastes it, since the API cannot add a named snippet ([`components-and-code.md`](components-and-code.md)).
  What the template accepts (2026-10-14):
  - `{{Field}}`, and `{{Field | json}}` for a quoted, escaped string. Field names with spaces work
    as typed (`{{Meta Description | json}}`), and a renamed field keeps the binding.
  - A field of a referenced item takes a dot: `{{Author.Name | json}}`. The underscore form in
    Framer's help article (`Author_Name`) stays raw text and breaks the JSON.
  - The editor lists the supported variables under the code box. For a reference it offers the
    item's text fields, slug and dates, and none of its link fields, so a referenced author's
    profile links are written into the snippet by hand, or kept as plain-text fields.
  - An image field gives its URL with `?width=1200&height=630`; a date field gives
    `2026-10-12T00:00:00.000Z`.
- **A meta description field.** Give a collection with detail pages a required "Meta Description"
  field, 155 characters at most, and set the template's page description to it. Without one, the
  description is whatever field is mapped, often an intro paragraph twice as long as search
  results show (2026-10-14).
- **Dead URLs on a live site.** Only for a site that is published to its production domain, never
  for branches, previews or test projects: when a CMS item's slug is renamed, a page is removed, or
  an item is duplicated and published by mistake (its slug ends in `-copy`), the old URL answers
  404 and Framer adds no redirect. Mention it, and offer a redirect in site settings (2026-10-14).
- **A new section, after its first publish.** When a site gains a section such as a blog, one line
  to the user is enough: Search Console re-reads the sitemap Framer generates on its own schedule,
  days later in one case, so resubmitting it and requesting indexing gets the section seen
  sooner. Importing the site into Bing Webmaster Tools from Search Console covers Bing, which
  ChatGPT search and Copilot draw on (2026-10-13).

## Speed

- **One variable font file.** Each text style on a static cut of a font loads its own file, about
  35 KB each. When the font has a variable cut and the site loads several static weights of it,
  suggest putting every style on the variable cut with its weight on the `wght` axis
  (`fontVariationAxes.wght`); one file then serves every weight. Three things quietly add files
  back: a code component that asks for the static family (use the variable family and set
  `fontVariationSettings: '"wght" N'` with each `fontWeight`), a blockquote style with no font,
  which falls back to Inter and can only be set in the editor, and one stray weight on a single
  layer. Say nothing when the site already loads one file per family (2026-10-14).
- **The largest text on the first screen.** An appear effect that starts at opacity 0 keeps that
  text out of the browser's Largest Contentful Paint until the effect plays: about 7 s on a phone
  test where the page painted in 1.5 s without it. Starting the same fade at opacity 0.1 instead,
  or even 0.01, lets the browser count the text at first paint (checked in Chromium, 2026-10-15).
  So suggest either a start above 0 or no effect on that one layer; everything around it can still
  animate. See also the static opacity trap in [`dsl-and-nodes.md`](dsl-and-nodes.md) (2026-10-14).
- **Scripts in custom code.** Analytics, chat widgets and embeds pasted into custom code are the
  only blocking JavaScript the site owner controls, and an analytics bundle was the largest single
  item on a phone test. If speed comes up, suggest starting such a script on the visitor's first
  interaction or a few seconds after load. Very short visits then go uncounted, so it is the
  user's call, and nothing changes without their yes (2026-10-14).
- **Framer's floor.** React, `framer.mjs`, `motion.mjs`, inline CSS for every breakpoint,
  hydration and the editor bar loader (`framer.com/edit/init.mjs`, loaded for every visitor) are
  Framer's and no setting removes them. Around 90 on PageSpeed's phone test is a good result for a
  Framer site, and 100 is out of reach; say so before a user sets that target (2026-10-14).

## Analytics

- **Analytics pasted into custom code must count page changes.** Framer moves between pages
  without a full page load, so a snippet that counts a pageview on load counts only the first page
  of each visit. Turn on the tool's own history-change tracking (PostHog: `defaults: '2025-05-24'`
  in `posthog.init`, which sets `capture_pageview: 'history_change'`). Leave the snippet's Run
  setting on "Once" ([`components-and-code.md`](components-and-code.md)). Before the fix, one site
  showed 178 pageviews in Framer's analytics against 56 in PostHog for the same days. Framer's
  built-in Google Analytics field in site settings is a separate route and not covered here
  (2026-10-12).
- **Framer's own analytics, for the agent.** When the user asks for traffic figures,
  `framer.agent.queryAnalytics({ query, from, to })` takes SQL over Framer's visit events; read the
  guide with `framer.agent.readProject([{ type: "implementation-guide-from-index", name: "Analytics" }])`.
  Filter on the live hostname: `*.framer.app` preview hosts carry your own checks (2026-10-12).

## Measuring speed, only when the user asks

A PageSpeed run takes minutes and a lot of tokens, so never measure on your own as part of a task.
When the user asks:

- The score the user sees is PageSpeed Insights (pagespeed.web.dev, or its API, which has a small
  daily quota without a key). Its phone score is simulated: it records the page on a fast
  network, then charges every request that finished before a paint to that paint.
- Lighthouse run from a cloud container, through its proxy, reads Framer pages as roughly twice as
  fast on FCP and LCP as PageSpeed reports, because the fonts and Framer's scripts arrive after the
  first paint there. Compare runs of one tool with each other, never across tools (2026-10-14).
- Google ranks on field data from real Chrome visitors (CrUX). A small site often has none yet,
  and then its lab score carries no ranking weight; it still matters to visitors.
