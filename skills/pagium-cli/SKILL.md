---
name: pagium-cli
description: Interact with a Pagium vault via the pagium CLI — read, create, save, search, list notes; query backlinks and broken links; import markdown; render HTML to plain text; watch for changes. JSON/NDJSON output by default for agents. Use when working with a Pagium vault from the command line, automating vault operations, or building agents against Pagium.
---

# Pagium CLI Skill

Use the `pagium` CLI to read, write, search, and observe a Pagium vault. The CLI is standalone — it does NOT require the desktop app to be running. Every invocation opens the vault directly (SQLite WAL mode; safe to run concurrently with the GUI).

The CLI is agent-first: default output is JSON (single object) or NDJSON (one JSON object per line for lists/streams). Use `--plain` for human-readable output. Use `--json` if you need to force JSON in a script that might inherit `--plain` from a config.

## Command reference

Run `pagium --help` to see all commands. `pagium <subcommand> --help` shows per-command flags. Full CLI spec: [pagium-cli design doc](https://github.com/inspirehub-labs/pagium/blob/main/docs/specs/2026-04-17-pagium-cli-design.md).

## Vault Targeting

Precedence (first hit wins):

1. `--vault <path>` flag (recommended for agents — explicit, deterministic)
2. `$PAGIUM_VAULT` env var
3. Upward walk from cwd for a `.pagium/` directory (git-style; good for humans in a vault)
4. Shared settings file (what the GUI's "last vault" points at)
5. Error: `no vault configured; run pagium open <path>` (exit code 1)

For agents, **always pass `--vault`**. Don't rely on implicit resolution.

```bash
pagium --vault ~/notes list
PAGIUM_VAULT=~/notes pagium list
```

## Slug Form

Most commands take a `<slug>` that accepts three forms; they all normalize to the same canonical vault-relative path:

| Input | Resolves to |
|-------|-------------|
| `my-note` | `my-note.html` |
| `projects/foo` | `projects/foo.html` |
| `my-note.html` | `my-note.html` |

Prefer the bare form (`my-note`) — it's what `pagium list` and `pagium search` emit.

## Common Patterns

```bash
# List everything; NDJSON — one note per line
pagium --vault ~/notes list

# Read a note as structured JSON (meta + body_html + parsed links)
pagium --vault ~/notes read my-note

# Full-text search (FTS5 with highlighted snippets)
pagium --vault ~/notes search "sqlite wal" --limit 10

# Create from stdin
echo '<p>Hello.</p>' | pagium --vault ~/notes create my-note --title "My Note"

# Transform a note in place
pagium --vault ~/notes read my-note \
  | jq -r .body_html \
  | your-transform \
  | pagium --vault ~/notes save my-note

# Link graph
pagium --vault ~/notes links backlinks my-note   # who links to this
pagium --vault ~/notes links outgoing my-note    # what this links to
pagium --vault ~/notes links broken              # all dangling targets

# Render an HTML note to plain text (for LLM context)
pagium --vault ~/notes render my-note

# Import a markdown vault
pagium --vault ~/notes import ~/old-obsidian-vault --dry-run
pagium --vault ~/notes import ~/old-obsidian-vault

# Watch for changes — long-lived NDJSON event stream
pagium --vault ~/notes watch --initial
```

See [COMMANDS.md](references/COMMANDS.md) for the full command surface with every flag, and [OUTPUT.md](references/OUTPUT.md) for JSON/NDJSON shapes and exit codes.

## Workflow: Agent Reading a Note

```bash
pagium --vault "$VAULT" read "$SLUG"
```

Returns:

```json
{
  "slug": "foo",
  "title": "Foo",
  "meta": {"version":1,"tags":["rust"],"created":"...","modified":"..."},
  "body_html": "<p>...</p>",
  "links": [{"target":"bar","text":"bar"}]
}
```

- `body_html` is the raw body — use this if you want to do further HTML processing.
- If you want plain text for an LLM prompt, use `pagium render <slug>` instead; it strips structure and emits a compact form. Cheaper tokens.
- `links` is the parsed outgoing-link list; use it to follow the graph without a second call.

## Workflow: Agent Writing a Note

`create` — new note. Body is read from stdin when piped; otherwise uses `--body-file`, otherwise writes an empty body.

```bash
echo '<p>First draft.</p>' | pagium --vault "$VAULT" create new-slug --title "New Slug"
```

`save` — overwrite body of an existing note. Preserves `pagium:title`, `pagium:created`, `pagium:tags`, and any custom `pagium:*` tags; refreshes `pagium:modified`. Requires stdin or `--body-file` (errors with exit 3 otherwise).

```bash
echo '<p>Updated body.</p>' | pagium --vault "$VAULT" save existing-slug
```

The round-trip guarantee holds: everything except `pagium:modified` is preserved byte-for-byte. Agents can safely pipe `read | transform | save`.

## Workflow: Agent Observing a Vault

For long-running agents that need to react to file changes:

```bash
pagium --vault "$VAULT" watch --initial --initial-limit 0
```

- `--initial` emits a synthetic `created` event for every existing note before going live.
- `--initial-limit 0` disables the cap (default 10000). Use on huge vaults only if you actually need all of them.
- A `{"event":"snapshot_complete",...}` sentinel separates the initial burst from live events — watch for it.
- Events: `created`, `modified`, `deleted`, `renamed`. See [OUTPUT.md](references/OUTPUT.md) for shapes.
- `watch` reads from the filesystem, not the DB — no contention with concurrent writers.

## Workflow: Bulk Import

Convert a markdown directory (Obsidian vault, collection of `.md` files, etc.) into Pagium HTML:

```bash
# Preview what would happen — no writes
pagium --vault "$VAULT" import ~/old-notes --dry-run

# Run it
pagium --vault "$VAULT" import ~/old-notes

# Strict mode — abort on anything lossy (e.g., ![[embed]] syntax)
pagium --vault "$VAULT" import ~/old-notes --fail-on-unsupported
```

- Wikilinks `[[target]]` and `[[target|display]]` → `<pagium-link target="target">display</pagium-link>`.
- YAML frontmatter → `pagium:*` meta tags (`title`, `tags`, `created`, `modified` map directly; custom keys → `pagium:custom:<name>`).
- `![[embed]]` has no Pagium equivalent yet; default behavior emits a `<pagium-link>` with a warning. Use `--fail-on-unsupported` when lossy conversion is unacceptable.
- Output mirrors input tree (`posts/foo.md` → `posts/foo.html`).
- Existing files are skipped with a warning; use `--force` to overwrite.

## Output Modes

| Flag | Effect |
|------|--------|
| (default) | JSON object for single-result commands, NDJSON for lists and streams |
| `--json` | Force JSON (useful if a caller passed `--plain`) |
| `--plain` | Human-readable: tables, colored text, compact summaries. **Not machine-parseable.** |
| `-v` / `--verbose` | Debug logging to stderr |
| `-q` / `--quiet` | Errors only |

**Agents: never rely on `--plain`.** Parse JSON. `--plain` is for humans only and may change cosmetically between versions.

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Usage error (bad args, no vault configured) |
| 2 | Not found (slug doesn't exist, vault path doesn't exist) |
| 3 | Conflict (slug already exists on `create`; DB locked past busy-timeout; `save` with no stdin/file) |
| 4 | I/O error (disk full, permission denied) |
| 5 | Parse / format error (malformed HTML, round-trip check failed) |

Errors write an envelope to stderr:

```json
{"error":"not_found","message":"no note at slug 'foo'","code":2}
```

Under `--plain`, errors print as `error: no note at slug 'foo'` to stderr. Exit code is set either way.

## Concurrency

- Every CLI invocation opens SQLite in WAL mode with a 15-second busy-timeout.
- Safe to run concurrently with the desktop app. The GUI's file watcher (150ms debounce) re-indexes anything the CLI writes, but that's idempotent — no divergence.
- Override the timeout for fail-fast behavior: `--timeout 500` (ms).
- `watch` does NOT open the DB; it only observes the filesystem. Zero contention regardless of how many GUIs or CLIs are also running.

## Installation Check

```bash
pagium version
```

Prints the binary version and the `pagium-core` schema version. Agents can gate on schema version when writing notes that use new format features.

If the command is not found, the CLI is not installed. Install options:

- Desktop app: Settings → "Install CLI" (creates a symlink in `/usr/local/bin`)
- Homebrew: `brew install inspirehub-labs/pagium/pagium-cli`
- Curl: `curl -fsSL https://pagium.app/install.sh | sh`

## Using With the `pagium-html` Skill

`pagium-cli` is the operational half; [`pagium-html`](../pagium-html/SKILL.md) is the format half. When an agent needs to write a note:

1. Consult `pagium-html` for the HTML body structure (meta tags, `<pagium-link>` syntax).
2. Use `pagium-cli` to pipe that HTML into `create` or `save`.

The CLI handles required-meta insertion and timestamps for you — you only need to supply the body HTML and the title. You almost never need to hand-write the full document frame.

## References

- [Pagium landing page](https://pagium.app)
- [CLI design spec](https://github.com/inspirehub-labs/pagium/blob/main/docs/specs/2026-04-17-pagium-cli-design.md)
- [Note format](../pagium-html/SKILL.md)
