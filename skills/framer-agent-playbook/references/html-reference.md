# Recreating an HTML reference

Traps from rebuilding a pixel-accurate HTML reference (a variant.com export) on the canvas
(2026-08-20).

- **Match the reference's render width before touching type sizes.** The `screenshot` + `url` query
  renders external URLs at 1200px. A display headline "not filling the column" is usually a
  breakpoint-width mismatch (1440 vs 1200), not a font-size problem.
- **Framer's font rendering is not the browser's.** Instrument Serif came out about 6% tighter on the
  canvas; px values from the reference CSS can need a bump (160px to 168px) to fill the same box.
- **`textWrapBalance` fights fidelity.** It re-rags lines; leave it off when matching exact breaks.
- **Column ratios lie.** `5fr`/`7fr` is not Tailwind's `col-span-5/7` with a gap. When line wraps
  must match, compute the real 12-column px widths and set those.
- **Seed the randomness.** Generated decorative sets (a 60-bar chart) should use a seeded PRNG in the
  exec script so a re-run reproduces the same layout.
