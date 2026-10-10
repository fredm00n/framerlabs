# Images and assets

- **Upload a local file with a base64 data URL.** `framer.uploadImage({ image:
  "data:image/png;base64,…", name, altText })` works for PNG and SVG and returns `{ url }` on
  `framerusercontent.com`.
- **Upload a video with a Node `Buffer`.** `framer.uploadFile({ file: { bytes:
  fs.readFileSync(path), mimeType: "video/mp4" }, name })` works. A data URL, a `Uint8Array` built
  in the script and `Uint8Array.from(buf)` all fail the type check, because the sandbox's
  `Uint8Array` comes from another realm; the `Buffer` returned by `fs` passes (2026-10-05). Read the files from `os.tmpdir()`, see
  [Running exec scripts](exec-scripts.md).
- **A full-screen component in a `FixedOverlayNode` covers the backdrop, so "click outside to
  close" stops working.** Let clicks through instead: `pointerEvents="none"` on the instance and on
  every layer inside it that fills empty space, `pointerEvents="auto"` on the buttons and on whatever must stay clickable. A code
  component's own wrapper also needs `none`, with `pointer-events: auto` set in its code on the part
  that should react. Checked with `elementFromPoint` and taps at 1440, 810 and 390 on a client's video
  overlay, 2026-10-08.
- **A full-screen overlay card must be `100%`, not `100vh`.** In a `FixedOverlayNode` the
  instance's `100vh` is the large viewport, so Android Chrome's bottom bar covered the card's
  buttons. `height="100%"` follows the overlay layer, which publishes as `position: fixed; top: 0;
  bottom: 0` and so tracks the visible screen. Make long content scroll inside the card: an
  absolute `overflow="auto"` frame holding a `1fr` spacer (with a `minHeight`) and the text (2026-10-07).
- **A `hoverEffect.backgroundColor` set through the API is invisible in Framer's Hover effect panel**
  (it lists opacity, scale, rotate, skew and offset only). It still works on the published site.
- **A colour token in `hoverEffect.backgroundColor` needs its fallback: `var(--token-<id>, rgb(…))`.**
  A published page only defines the CSS variables of tokens it already uses elsewhere. Written
  bare, a token that appears nowhere else on the page resolves to transparent, so a black button
  with a white label vanished on hover. Fills set through the normal attributes get the fallback
  from Framer; hover values written through the DSL do not (2026-10-07).
- **`hoverEffect.transition` drops `spring-physics` silently** and keeps the previous value.
  Variants take `spring-physics 400 30 1 0s`; for a hover effect use the equivalent
  `spring-duration 0.3s 0.25 0s` (stiffness 400 gives about 0.31s, damping 30 a ratio of 0.75).
  Sampled live on a client site, it matched a variant hover with the physics spring frame for frame (2026-10-07).
- **A gradient fill keeps at most 10 stops.** From the eleventh stop on, Framer rewrote each colour
  as the second stop's colour with a digit appended (`rgba(0, 0, 0, 0.013)0`), and the browser
  dropped the whole gradient. An eased fade with 9 stops is smooth enough (2026-10-07).
- **An instance pinned top and bottom inside a `layout="null"` frame takes the component's own
  height.** With `height="auto"` the bottom pin is kept in the serialized attributes, but the
  instance renders at the primary variant's fixed height. Make the variant height match the frame (2026-10-07).
- **Variables created in the same call as their `+ComponentNode` get no `renamedIds` entry.**
  The component and its primary variant do. Read the variable ids back with
  `serialize({ id: componentId, depth: 0 }).variables` before binding them (2026-10-07).
- **`getBranchChanges` needs the branch id as a string argument.** Called bare, it throws a typia
  assert error (2026-10-07).
- **A one-variant-per-item component gives a native hover switcher.** Give each row
  `onMouseEnter` and `onTap` `SET_VARIANT` to its own variant on the primary; it propagates to
  variants created before or after. Stack the detail panels absolute and change only `opacity`,
  pins and `pointerEvents` per variant, and the variant transition animates the swap. Hidden panels
  need `pointerEvents="none"` or they block the links of the visible one (2026-10-07).
- **The image focal point is not reachable from the API.** `ImageAsset` exposes only `altText` and
  `resolution` (`cloneWithAttributes` takes nothing else), and `fillImagePositionX/Y` is dropped
  without an error when the frame's `fill` is bound to an image variable. Set it by hand in the
  editor (image panel, Focal point), or upload a second crop made for the other shape. The hand route
  keeps one photo per person (2026-10-07).
- **`uploadFile` keeps a photo byte for byte**, so it is the route for a "download full-res"
  link: a 10.7MB JPG came back at `framerusercontent.com/assets/<hash>.jpg` with the same length.
  The asset is served with no `Content-Disposition`, so the link opens the image in a tab rather
  than saving it; cross-origin, the HTML `download` attribute would not help either (2026-10-07).
- **Generated art is cheap and on-brand.** Three routes have worked: procedural dithered scenes
  drawn with numpy (no photos), two-colour Bayer dithers of real photos with ffmpeg (`paletteuse`
  with a 16×16 palette PNG), and recoloured CC0 SVG illustrations.
- **`fillImagePositionX` moves the crop window, not the image.** `left` shows the left part of the
  picture, which feels reversed when the subject is off-centre.
- **Set `altText` with one `SET` per `applyChanges` call.** On 2026-09-29 five `SET <id> altText="…"`
  lines in one call all landed on the first id, each overwriting the last, and the other four nodes
  kept no alt text. One call per node worked. Read the value back after writing it.
- **Image Tile (Fill > Type: Tile) cannot be set through the API.** `fillImageFit`, `fillType`
  and every similar DSL attribute or `setAttributes` key are ignored without an error, and the
  setting does not appear in `serialize`. Have the user set it by hand on one frame, then `DUPE` that
  frame: the copy keeps Tile, and so does a later `SET <id> fill="<url>"`. At Scale 100% a tile is
  drawn at half the image's pixel size, so upload at twice the tile width you want (2026-10-06).
- **A fine repeating pattern baked into an image (scanlines, a dense grid) shows moiré** once Framer
  scales it to a width that is not a whole multiple. It showed on a scanline map (2026-10-08). Flat fills with an outline survive any size; if a pattern is the point, make it
  coarse (several CSS pixels per period) or draw it natively.
- **Uploading an SVG strips a `viewBox` that matches its `width` and `height`**, and an SVG image
  without a `viewBox` does not scale: drawn into a smaller frame it is cropped, not shrunk. Give the
  file `width` and `height` at half the `viewBox` size before upload, so the `viewBox` survives.
  The upload also inlines classes into `fill` attributes, so recolour a logo by replacing its path
  fills, not with a `fill` on the root (2026-10-09).
- **A wide SVG blur is cheaper and steadier as a PNG.** A 46-unit `feGaussianBlur` on a large glow
  renders faint rings in Chromium; rasterise it once with transparency
  and use the PNG as the fill.
- **Framer smooths images scaled to a non-integer size.** Pixel-art and dither grain survives if the
  file is about twice the displayed size.

- **`framer.uploadImage` refuses `{ bytes, mimeType }` built in the exec sandbox** (2026-10-08).
