# Pagium CLI Command Reference

Complete command surface. Run `pagium <cmd> --help` for the authoritative flag list.

Global flags (apply to every command):

| Flag | Env | Default |
|------|-----|---------|
| `--vault <path>` | `PAGIUM_VAULT` | implicit resolution (see SKILL.md) |
| `--json` | — | on by default |
| `--plain` | — | off |
| `-v, --verbose` | `PAGIUM_LOG=debug` | off |
| `-q, --quiet` | — | off |
| `--timeout <ms>` | — | 15000 |

## Vault Meta

### `pagium open <path>`
Persist `<path>` as the shared-settings "last vault." Humans run this once; agents pass `--vault` instead.

### `pagium status`
Current vault, note count, last-indexed time, schema version. Single JSON object.

### `pagium version`
Binary version + `pagium-core` schema version. Single JSON object.

## Notes (root)

### `pagium list [--dir <subdir>]`
List every note in the vault (or only under `<subdir>`). NDJSON, one note per line.

```json
{"path":"projects/foo.html","slug":"projects/foo","title":"Foo","created":"...","modified":"...","tags":["rust"]}
```

### `pagium read <slug>`
Parse one note; return structured JSON (meta + body HTML + outgoing links). Single JSON object.

### `pagium create <slug> [--title T] [--body-file F]`
Create a new `.html` file in the vault with required meta tags. Body comes from stdin if piped; else `--body-file`; else empty body. `--title` defaults to the last slug segment if omitted. Conflict (slug exists) → exit code 3.

### `pagium save <slug> [--body-file F]`
Overwrite an existing note's body. Preserves every `pagium:*` meta tag except `pagium:modified`. Requires stdin or `--body-file` — errors with exit 3 otherwise.

### `pagium delete <slug> [--yes]`
Delete the file and its index entries. Requires `--yes` (no interactive prompt; CLI is non-interactive).

### `pagium search <query> [--limit N]`
FTS5 full-text search. NDJSON, one match per line, with highlighted snippet.

```json
{"slug":"foo","title":"Foo","snippet":"...has <mark>rust</mark> in it..."}
```

### `pagium notes rename <slug> --title T`
Update the note's `pagium:title` meta tag (does not rename the file). Single JSON object.

### `pagium notes duplicate <slug>`
Copy `<slug>` to `<slug>-copy`, refreshing timestamps.

### `pagium notes move <slug> <dest-path>`
Move/rename the file; rewrites the index. `<dest-path>` is a new slug (bare or with subdirectories).

## Links

### `pagium links backlinks <slug>`
Every note that contains a `<pagium-link target="<slug>">`. NDJSON.

### `pagium links outgoing <slug>`
Every `<pagium-link>` inside `<slug>`. NDJSON. Derived client-side from a parse — one I/O.

### `pagium links broken`
Every `<pagium-link target>` in the vault that doesn't resolve to a file. NDJSON.

## Folders

### `pagium folders list [--dir <subdir>]`
Enumerate subdirectories of `<subdir>` (or vault root). NDJSON.

### `pagium folders create <path>`
Create an empty subdirectory.

### `pagium folders rename <old> <new>`
Rename a subdirectory. Slugs of notes inside are updated in the index.

### `pagium folders delete <path> [--yes]`
Delete a subdirectory and everything under it. Requires `--yes`.

## Favorites

Favorites live in the shared GUI settings file, scoped per vault.

### `pagium favorites list`
NDJSON of favorited slugs for the current vault.

### `pagium favorites add <slug>` / `pagium favorites remove <slug>`
Toggle. No-op if already in the desired state.

## Agent Extras

### `pagium render <slug>`
Strip `<head>`, walk `<body>`, emit plain text. **Output is raw text, not JSON.** Use this when feeding a note to an LLM — cheaper tokens than `body_html`.

Supported body elements:
- Headings → `#` / `##`
- Paragraphs, lists (nested)
- Code fences with language extraction
- Inline bold/italic
- `<pagium-link target="X">` → `[[X]]`

Degraded rendering:
- Tables → `[table: N rows x M cols]` placeholder (GFM table rendering is v1.1)
- Images → `![alt](src)` passthrough

### `pagium import <md-dir> [--dry-run] [--force] [--fail-on-unsupported]`
Convert a markdown directory into Pagium HTML:
- `pulldown-cmark` → HTML body
- YAML frontmatter → `pagium:*` meta tags
- `[[wikilinks]]` → `<pagium-link>`
- `![[embed]]` → `<pagium-link>` with a warning, OR abort with `--fail-on-unsupported`

Output shape (NDJSON per file):

```json
{"source":"foo.md","target":"foo.html","status":"created","warnings":[]}
```

Conflict policy: skip with warning; `--force` overwrites.

### `pagium stats [--tags] [--orphans]`
Default (no flags) — single JSON object:

```json
{"notes":142,"links":387,"broken":3,"orphans":12,"tags":24}
```

`--tags` NDJSON: `{"tag":"rust","count":42}` per line.
`--orphans` NDJSON: notes with zero backlinks AND zero outgoing links.

### `pagium watch [--filter <substr>] [--initial] [--initial-limit N]`
Long-lived NDJSON event stream:

```json
{"event":"created","slug":"foo","path":"foo.html","at":"..."}
{"event":"modified","slug":"foo","path":"foo.html","at":"..."}
{"event":"deleted","slug":"foo","path":"foo.html","at":"..."}
{"event":"renamed","slug":"bar","from":"foo.html","to":"bar.html","at":"..."}
```

- `--filter "projects/"` — only emit events for paths containing the substring.
- `--initial` — emit a synthetic `created` event for every existing note before going live.
- `--initial-limit N` — cap the initial burst at N (default 10000; `0` = unlimited).
- After the initial burst: `{"event":"snapshot_complete","at":"..."}` sentinel, then live events.

Does NOT open the DB — pure filesystem observer. Safe under any number of concurrent writers.

### `pagium reindex`
Rebuild the FTS5 index from the HTML files on disk. Useful after `git checkout` or manual file shuffling. Single JSON object summary.

## GUI-Only (NOT in the CLI)

These exist as desktop app features but have no CLI equivalent:

- **Reveal in Finder** — macOS GUI affordance
- **Open in new window** — multi-window GUI chrome
- **Plugin reload / DevTools / screenshots** — Pagium has no plugin system yet

If you want an agent to do plugin-like work, operate on note files directly — that's the whole point of HTML as the format.
