# Framer build checklist

What a native Framer build must have before anyone calls it done. Started 2026-10-03 from notes on a
client build; grow it after every build. The API traps behind each step live in the other files of
this skill (see [`SKILL.md`](../SKILL.md)), and the layout rules in [`breakpoints.md`](breakpoints.md).

Tick each line in the build's own notes, or write why it does not apply.

## Before building

- [ ] The user named the Framer project. The API key stays in the session, never in the repo or memory.
- [ ] Nothing is deleted without the user's word (see [`destructive-operations.md`](destructive-operations.md)
      on irreversible deletes).
- [ ] Every image's licence checked. **No CC BY-NC, no "in copyright"** on a client site: a
      practice's website is commercial use. CC BY needs a credit and a note that the image was
      changed (toning counts).
- [ ] When the build ports an HTML mockup, its notes were read for anything marked "for the fact check" or "Framer port".
- [ ] The site's Framer plan checked before creating anything it caps, CMS collections first. Count
      what the file already has against the plan's limit; with no room left, fit the content into
      an existing collection or ask the user. Deleting a collection to free a slot takes their
      word (2026-10-06).
- [ ] What the agent cannot do was listed for the user at the start, so it is not discovered at the end:
      upload a font, set a form's destination, resize a vector shape (workarounds in
      [`dsl-and-nodes.md`](dsl-and-nodes.md)), and see scroll effects or overlays in a
      screenshot. The user does the first two in the editor; the last two are checked in the preview
      or on the published page (2026-10-06).

## In a client's existing file

Rules from a client page rebuild, 2026-10-05.
When the file is the client's live site, [`live-files.md`](live-files.md)
applies first: branches, no tests in the file, merge or publish only when the user asks.

- [ ] Existing components, text styles and colour tokens were searched first and reused wherever
      they fit. Something new was made only when nothing in the file already does the job, and it
      sits under a folder named after the client (components, text styles, colour tokens, code
      files) so the user can list it for the client.
- [ ] No shared component used on other pages, and no layout template, was edited. When a page
      needs a variant of one, duplicate it into the client folder instead.
- [ ] Sections being replaced on an existing page were hidden inside a backup frame, never deleted.
      They go only once the client signs off.
- [ ] The rest of the project was checked for side effects: a swapped shared element (a banner, a
      card) was checked on every page that uses it, at all three breakpoints.
- [ ] Mobile motion was built and checked, not left to the desktop defaults: every new component
      has a stacked Tablet/Phone variant set with its own taps, and every instance picks it per
      breakpoint.

## Foundations (once per site)

- [ ] Breakpoints Desktop 1200 (designed at 1440), Tablet 810, Phone 390, on the layout template and
      every page.
- [ ] Colour tokens, named by role, in folders: `Ink/`, `Ground/`, `Line/`, `Accent/`. No raw hex
      on the canvas.
- [ ] Text styles for every text on the site, in folders by role (`Display/`, `Heading/`, `Body/`,
      `Label/`), each with its tablet and phone sizes set at creation. The tag (h1, h2, p) is set
      on the style, so the outline comes out right. No inline `fontSize` anywhere.
- [ ] One type scale, not one style per use. Inside each folder, name styles by size (`Heading/XL`,
      `L`, `M`, `S`, `XS`; `Body/XL` down to `Body/XS`), not by where they sit (`Heading/Pillar`,
      `Heading/Apply`). Two styles within 2px of each other are one style. A style that differs
      only in colour is not a new style: colour the text layer instead. One weight per tier:
      Thin for display sizes from 112px, ExtraLight for page titles and pull quotes, Light for
      headings, Regular for body. Hover underlines are the one allowed twin (`Body/Title Hover`).
      One site ended at 36 styles after 66 had grown page by page (2026-10-05).
- [ ] The nav marks the current page with its own indicator (a dot under the link, for example):
      a string variable on the nav component, bound to a layout template variable, set on each
      page. `link.current` only restyles text, so it cannot draw the dot (2026-10-04).
- [ ] No Gutter or Spacer layers: a 12-column split becomes `fr` widths plus a gap, and "push the
      button to the bottom" becomes `stackDistribution="space-between"` with the text grouped above it (2026-10-04).
- [ ] One layer per piece of content on every breakpoint, adapted with overrides, so the site
      owner edits each text once. Any tablet or phone copy is named for its breakpoints and noted
      with why no override could do it (see [`breakpoints.md`](breakpoints.md)) (2026-10-13).
- [ ] One layout template holding everything shared: nav, footer, smooth scroll, page transition.
      Every page uses it. Smooth scroll is a Lenis code module from framer.com/m.
- [ ] Components for everything used on more than one page: buttons and text CTAs, nav (Desktop,
      Tablet, Phone and an Open menu variant), footer (Desktop, Phone), cards, list rows, captions.
      Each with hover and pressed variants where it is clickable. A control that repeats inside
      other components (a play button, an arrow) is its own component with a variant per
      background and a hover twin of each, and the parent's hover variant switches it, so the
      whole card lights the control. For example `Media/Play Button` (On Ink, On Light, On Photo)
      inside a talk card and `Media/Film` (2026-10-05).
- [ ] Phone menu: every link in the open drawer also sets the closed variant on tap, so the menu
      closes when the visitor navigates (2026-10-04).
- [ ] Every form is a component with Default, Sending and Sent variants: the form dims and stops
      taking input while sending, then a thank-you note in the client's own words shows where the
      visitor is looking (beside the button on a long form). The submit button has its own
      Pending, Success and Error variants, mapped with `formButtonPendingVariant` and its siblings
      once the form carries `formSubmitButtonId` (see [`forms-and-webhooks.md`](forms-and-webhooks.md)).
- [ ] Form inputs: radios checked as a 5 to 6px ink border on a white fill (a dot), checkboxes and
      radios unchecked by default (`formBooleanInputValue="false"`), focus border in ink, never
      Framer's default blue.
- [ ] Every CMS video comes as two fields: a File field for an upload and a URL field as the
      fallback, bound so an uploaded file wins and an empty one falls back to the URL. Heavy videos
      can then move to an external host such as Cloudflare without a rebuild
      ([`cms.md`](cms.md) has the binding) (2026-10-06).
- [ ] A row of CMS items that slides sideways is a Collection List that scrolls horizontally, with
      a drag-to-scroll code override for desktop mice. The native Slideshow
      and Carousel cannot put each CMS item on its own slide: a list in the slot becomes one slide
      holding every item (2026-10-06).
- [ ] Code only where native layers cannot do it, each piece named in the build notes with the
      reason.

## Every page

- [ ] Built at Desktop first, then Tablet and Phone section by section, not at the end.
- [ ] Exactly one H1. H2 for sections, H3 inside them, no skipped levels. Visual size comes from the
      style, never from picking a lower or higher tag.
- [ ] Every image has real alt text, written for someone who cannot see it; decorative images
      have empty alt.
- [ ] Page title, under 60 characters, and meta description, 140 to 160 characters, in the
      client's own words.
- [ ] Social image, 1200 x 630, per page where it differs from the site default.
- [ ] Links: internal ones point at real pages, external ones at the right URL (no tracking junk),
      `mailto:` and booking links checked by hand.
- [ ] Appear effects play once (`replay="false"`), no exit states. Headings move, paragraphs fade
      at most. The largest text on the first screen stays still or starts its fade above opacity 0
      ([`seo-and-speed.md`](seo-and-speed.md)).
- [ ] Scale effects only scale down. The layer is built at its largest size and the effect shrinks
      it; a layer scaled up blurs once the browser rasterises it, vectors included. Static `scale`
      stays at 1, because the effect multiplies it (2026-10-06).
- [ ] Anything that grows or shrinks vertically animates its height (2026-10-02). FAQ lists,
      accordions and forms sit in a stack with a Flow effect, bounce 0, 0.5 to 0.6s
      (`flowEffect.transition="spring-duration 0.6s 0 0s"`) (2026-10-04).
- [ ] Every "contact" or "apply" link lands at the top of the contact page, not at the form: the
      intro text there matters. The hero and the nav never send visitors to an off-site booking
      tool; the booking link lives on the contact page (2026-10-04).
- [ ] A list's closing border uses the same colour as its row borders. Check every list, light and
      dark sections both (2026-10-04).
- [ ] Sizes match the approved design per page: when one page's version of a shared style
      is smaller, it takes the next step down the scale (`Heading/M` instead of `Heading/L`),
      never an inline size and never a new near-duplicate style.
- [ ] Inner pages share one page-title style and start at the same height: the H1's box sits
      150px under the nav bar on Desktop, 112px on Tablet and 72px on Phone (hero top padding =
      nav height + that gap). Hero layouts may differ; the title style and that gap do not
      (2026-10-05).
- [ ] Scroll-scrubbed animations start when the section's middle reaches the viewport bottom and
      finish 100 to 200px after its bottom reaches the viewport bottom (2026-10-02).
- [ ] Copy matches the approved design word for word; every visible line traceable to the
      client's own site or the user.
- [ ] The canvas is tidy: Desktop, Tablet and Phone side by side from x 0, and the ground nodes the
      page links to (slideshow slots, overlay targets) on their left, in the order they appear on
      the page (2026-10-06).

## Site-wide

- [ ] Site title, description, favicon (light and dark), apple touch icon and default social image
      on the root.
- [ ] 404 page on the layout template, plain words, a way back home.
- [ ] Privacy page when any form collects personal data.
- [ ] Site language set in Site Settings, Localization (the API has no setter; `<html lang>` ships
      empty otherwise).
- [ ] Forms send somewhere real, with a success state, and someone has tested one submission.
- [ ] Cookie banner if any analytics or embeds set cookies in a jurisdiction that needs consent.
- [ ] Structured data where it helps (for a practice: `Physician` or `MedicalBusiness`, `sameAs`
      links to the client's profiles), through the page's custom code if needed.
- [ ] Pitch previews are `noIndex` per page. Remove it at launch.
- [ ] The "Made in Framer" badge hidden by adding
      `<style>#__framer-badge-container { display: none !important; }</style>` to the site's custom
      code. Framer's "Edit Content" button shows to every visitor on a framer.app
      address only; on a custom domain only collaborators see it, and there is no switch for it.

## Launch

- [ ] Redirect map from the old site: every URL in its sitemap, every URL search engines know
      (`site:` search), and every URL the crawl found linked, each mapped to its new page or to the
      closest one. Built as Framer redirects (308) before the domain moves. Kept in the build's
      `framer/redirects.md`.
- [ ] Old URLs that answered 404 or 410 are left alone unless something links to them.
- [ ] Custom domain connected, `www` and apex both resolving, HTTPS on.
- [ ] `noIndex` removed, sitemap reachable, Search Console told about the move.
- [ ] Analytics checked on the live domain.

## QA before the user sees it

- [ ] Before a client sees new pages, an independent fact check: a subagent with no project
      context (told not to open the repo) extracts every claim from the published pages, alt text
      and form settings included, and grades each against the client's live site and their public
      profiles (VERBATIM, SUPPORTED, PARTIAL, UNSUPPORTED, CONTRADICTED). Save the report in a
      `fact-check/` folder with the build's notes. Fix every PARTIAL and UNSUPPORTED line or
      take it to the user (2026-10-05).

- [ ] Published page checked with Playwright at 1440, 810 and 390, real wheel scrolling, at 1x and
      2x pixel ratio; also 2560 wide and a short 1280 x 720 window.
- [ ] Side by side with the approved design at the same widths.
- [ ] No horizontal overflow, no clipped headings, no text overflowing its box, no console errors.
      The "accelerometer is not allowed" and "devicemotion events are blocked" lines are not ours:
      they come from Framer's on-page editing bar, an iframe from `framer.com/edit` loaded on every
      published page.
- [ ] Every link and button clicked once on the published page, the phone menu included.
- [ ] After any publish or code-file change, every interactive component on the page tried again,
      not only the one edited: an id-based link can break in code nobody touched (2026-10-09).
- [ ] Reduced motion: nothing essential depends on animation.
- [ ] Lighthouse on the published page: performance, accessibility, SEO; images not oversized.
- [ ] The build's `framer/README.md` written: what is native, what is code, where Framer fell
      short, defaults picked without asking, and the build order that worked.
