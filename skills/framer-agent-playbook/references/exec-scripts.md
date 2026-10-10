# Running exec scripts

- **There is no `api` global in the `exec` VM.** The generic skill prompt mentions
  `api.extractDesignPatterns`, `api.search` and `api.serialize`; none of them exist. Use
  `framer.agent.*` (`serialize`, `getNodesOfTypes`, `getNode`, `readComponentControls`).
- **Write scripts to a file and pipe them** (`exec -s <id> < script.js`). Even a quoted heredoc
  breaks with `unexpected EOF` once the JS holds an odd number of single quotes, and inline scripts
  (or Bash heredocs) above about 8 KB get truncated on Windows. A small shared prefix file (tokens,
  image URLs, a `run(dsl)` helper that prints the result) concatenated in front of each script
  keeps them short (2026-09-22).
- **`fs` inside `exec` refuses relative paths** ("access outside allowed directories"), because the
  VM's working directory is not your shell's. Read and write under `require("os").tmpdir()` (2026-09-22).
- **`readProject` returns `{ results }`.** Font search results sit under `results[i].results`, a
  guide's text under `results[i].guide`. More than three or four queries in one call drops the relay
  ("Connection error", then empty output).
- **`applyChanges` errors are top-level** (`res.errors`, `res.warnings`, `res.linter`); there is no
  `res.commands`. A script that prints commands sees a clean run that failed. `res.errors` is an
  **object keyed by message** (each value lists the ids), not an array, so `errors.length` is
  undefined and misses every failure; test `Object.keys(res.errors).length` (2026-10-06).
- **`applyChanges` can time out at 120s on heavy pages while the write still lands.** Re-read before
  re-applying.
- **Temp ids do not survive across `applyChanges` calls.** Referencing `parent="heroWrap"` in a later
  call, where `heroWrap` was a temp id from an earlier one, silently drops the command: no parse
  error, no command error, and the created-node count still looks plausible. Carry the canonical id
  from `renamedIds`, or re-read the tree (2026-08-24).
- **Temp ids are global to the exec session.** With several agents on one session,
  `parent="needsIn"` resolves to whichever builder created that alias first and writes onto their
  page. Prefix every temp id per builder, and never let one temp id be a prefix of another
  (`navCta` next to `navCtaL` silently lost eight commands while `renamedIds` still listed them) (2026-09-17).
- **The same temp id twice in one call creates the first node and drops the second**, with "The
  requested id already exists". A section header `fH2` and a card heading `fH2` in one call lost the
  card heading. Prefix per section as well as per builder (2026-09-22).
- **Variable ids given in `+Variable <id>` are not kept.** Framer assigns its own and `renamedIds`
  does not list them. Serialize the scope (component or collection) to read the real ids.
- **`+ColorStyleTokenNode` returns UUID ids**, and `var(--token-<uuid>)` resolves fine.
- **A temporary id stays reserved for the whole exec session**, even after its node is deleted.
  Reusing one in a later call (`sbT`, `lbar`, `pcInst` after a `DEL`) fails with "The requested id
  already exists", every `SET` that names it goes to the old node, and a `DEL` in the same call still
  runs. Give every call its own prefixes and every `+` node a fresh id (2026-10-04 to 2026-10-09).
