# The Assets panel

Framer's "Asset Panel Refresh" (2026-10-26, framer.com/updates/asset-panel) lets one folder mix
components, colour styles, text styles, link styles, code files, layout templates and vectors.
What follows was read and written through the Framer connector (MCP) on a real client file.

- **One root folder takes every asset type.** The panel is a `VirtualFolderNode` with id
  `assetsEntityRoot`; its `referenceTypes` list the thirteen kinds it accepts. Framer's old fixed
  sections ("Components", "Text Styles", "Colors", "Layout Templates" and the rest) are ordinary
  folders inside it, with ids starting `asset-section-`, and folders nest (2026-10-30).
- **A folder holds references, not the assets.** Each entry is a `VirtualReferenceNode` whose
  `reference` is the asset's id. To file an asset, `tree.move` its reference into the folder; to
  file one that has none, insert a `VirtualReferenceNode` with `parent` set to the folder and
  `reference` set to the asset's id. Filing never renames the asset, and layers point at styles by
  id, so moving a text style between folders leaves every layer that uses it intact (2026-10-30).
- **An asset with no reference shows loose at the top of the panel.** Code files had none on one
  site and sat at the root under every folder. Assets created through the connector
  (`tree.insert` of a style, colour token and so on) also get no reference, so file each new one
  with a `VirtualReferenceNode` in the same script, or it lands loose at the root (2026-10-30).
- **Framer's built-in sections can be moved into a folder.** Once everything is filed into custom
  folders, the old sections stay behind empty and crowd the root. `tree.move` each
  `asset-section-…` folder (and the library folders Framer adds for outside components) into one
  folder at the bottom, so the root shows only your own folders. Removing a built-in section also
  works, but moving it is the reversible choice (2026-10-30).
- **Plan the folders by how the site is assembled.** A layout that worked for a client handover:
  Foundations (Colour, Type, Links), Elements (buttons, labels), Patterns (navigation, footer,
  accordions), Layout (templates), Code, Archive (the template's leftovers, filed instead of
  deleted), then the tucked-away built-in sections (2026-10-30).
