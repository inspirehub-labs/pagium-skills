# Pagium CLI Output Shapes

Every JSON shape an agent will encounter. Cardinality shown first so you know how to parse.

## Shape by Cardinality

| Command | Shape | Parse as |
|---------|-------|----------|
| `list`, `search`, `links backlinks/broken/outgoing`, `folders list`, `import`, `stats --tags`, `stats --orphans` | **NDJSON** | one JSON object per line |
| `read`, `status`, `version`, `stats` (default), `create`, `save`, `delete`, `open`, `reindex`, `notes rename/duplicate/move` | **Single JSON object** | one parse |
| `watch` | **NDJSON stream, long-lived** | one object per line, forever |
| `render` | **Plain text** | raw text; NOT JSON |
| all errors | **JSON on stderr**, exit code ≠ 0 | parse stderr on non-zero exit |

## Exit Codes

| Code | Meaning | Typical cause |
|------|---------|---------------|
| 0 | Success | — |
| 1 | Usage error | bad args; no vault configured |
| 2 | Not found | slug doesn't exist; vault path missing |
| 3 | Conflict | `create` on existing slug; DB locked past busy-timeout; `save` with no stdin and no `--body-file` |
| 4 | I/O error | disk full; permission denied |
| 5 | Parse / format error | malformed HTML; round-trip check failed |

## Error Envelope

Errors always write JSON to **stderr**:

```json
{"error":"not_found","message":"no note at slug 'foo'","code":2}
```

Under `--plain`, stderr becomes `error: no note at slug 'foo'`. The exit code is set either way. Agents: always inspect both the exit code and stderr.

## Shape Catalog

### `pagium list` — NDJSON

```json
{"path":"foo.html","slug":"foo","title":"Foo","created":"2026-04-17T10:30:00+00:00","modified":"2026-04-17T12:00:00+00:00","tags":["rust","pkm"]}
{"path":"projects/bar.html","slug":"projects/bar","title":"Bar","created":"...","modified":"...","tags":[]}
```

### `pagium read <slug>` — single JSON object

```json
{
  "slug": "foo",
  "title": "Foo",
  "meta": {
    "version": 1,
    "tags": ["rust"],
    "created": "2026-04-17T10:30:00+00:00",
    "modified": "2026-04-17T12:00:00+00:00"
  },
  "body_html": "<h1>Foo</h1><p>...</p>",
  "links": [
    {"target": "bar", "text": "bar"}
  ]
}
```

`meta` may contain extra keys for custom `pagium:*` tags on the note. `body_html` is the body content, not the whole document — the `<head>` is NOT included.

### `pagium search <query>` — NDJSON

```json
{"slug":"foo","title":"Foo","snippet":"...has <mark>rust</mark> in it..."}
{"slug":"bar","title":"Bar","snippet":"...<mark>rust</mark> and tauri..."}
```

`<mark>` wraps FTS5 hits in the snippet. Strip if you don't want the highlight.

### `pagium links backlinks <slug>` — NDJSON

```json
{"slug":"other-note","title":"Other Note","text":"see this"}
```

One record per `<pagium-link>` that targets `<slug>`. `text` is the inner text of the link (the display text).

### `pagium links outgoing <slug>` — NDJSON

```json
{"target":"bar","text":"bar","exists":true}
{"target":"missing-note","text":"broken","exists":false}
```

### `pagium links broken` — NDJSON

```json
{"source_slug":"foo","target":"missing-note","text":"broken"}
```

### `pagium stats` (default) — single JSON object

```json
{"notes":142,"links":387,"broken":3,"orphans":12,"tags":24}
```

### `pagium stats --tags` — NDJSON

```json
{"tag":"rust","count":42}
{"tag":"pkm","count":18}
```

### `pagium stats --orphans` — NDJSON

```json
{"slug":"orphan-1","title":"Orphan","modified":"..."}
```

### `pagium watch` — NDJSON stream

```json
{"event":"created","slug":"foo","path":"foo.html","at":"2026-04-17T12:05:00+00:00"}
{"event":"modified","slug":"foo","path":"foo.html","at":"2026-04-17T12:05:10+00:00"}
{"event":"deleted","slug":"foo","path":"foo.html","at":"2026-04-17T12:10:00+00:00"}
{"event":"renamed","slug":"bar","from":"foo.html","to":"bar.html","at":"2026-04-17T12:11:00+00:00"}
```

With `--initial`: the initial burst uses `"event":"created"` for every existing note, then:

```json
{"event":"snapshot_complete","at":"2026-04-17T12:04:59+00:00"}
```

— after this sentinel, subsequent events are live.

### `pagium import <md-dir>` — NDJSON

```json
{"source":"foo.md","target":"foo.html","status":"created","warnings":[]}
{"source":"bar.md","target":"bar.html","status":"skipped","warnings":["file exists; use --force to overwrite"]}
{"source":"baz.md","target":"baz.html","status":"created","warnings":["![[embed]] downgraded to <pagium-link>"]}
```

`status` ∈ `created`, `skipped`, `overwritten`, `failed`.

### `pagium render <slug>` — plain text

NOT JSON. Example output:

```
# Foo
Tags: rust, pkm

This paragraph mentions [[bar]] and has some **bold** text.

- A list
  - Nested item
- Another item

```rust
fn main() {}
```
```

### `pagium status` / `pagium version` — single JSON object

```json
{"vault":"/Users/alice/notes","notes":142,"schema_version":1,"last_indexed":"2026-04-17T12:00:00+00:00"}
```

```json
{"binary":"0.3.0","schema":1}
```

## Parsing Tips for Agents

1. **NDJSON:** split on `\n`, parse each non-empty line as JSON. Don't try to parse the whole stream as one array.
2. **Watch:** keep reading forever, one line at a time. Watch for `snapshot_complete` to know the initial burst is done.
3. **Render:** treat output as a string. No structure to parse.
4. **Errors:** always read stderr on non-zero exit. The JSON envelope tells you which error class (so you can map to exit-code semantics reliably).
5. **Locale-independence:** JSON timestamps are always ISO 8601 with explicit offset. The CLI doesn't localize anything.
