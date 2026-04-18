# Pagium Meta Tags Reference

All Pagium metadata lives in `<head>` as `<meta name="pagium:*">` tags. The parser preserves every `pagium:*` tag on round-trip; unknown names are allowed and retained.

## Core Meta Tags

| Name | Required | Format | Purpose |
|------|----------|--------|---------|
| `pagium:version` | Yes | Integer string (`"1"`) | Format version. Used for migrations. |
| `pagium:title` | Yes | HTML-escaped string | The display title shown in the sidebar and search results. |
| `pagium:created` | Yes | ISO 8601 UTC | Creation timestamp. Never mutated after initial write. |
| `pagium:modified` | Yes | ISO 8601 UTC | Last-save timestamp. Rewritten by the engine on every save. |
| `pagium:tags` | No | Comma-separated | Tag list. Example: `"rust,pkm,active"`. Whitespace around commas is trimmed. |

## Timestamp Format

Timestamps are ISO 8601 with explicit UTC offset. Both of these are accepted:

```
2026-04-18T12:00:00+00:00
2026-04-18T12:00:00Z
```

The engine emits the `+00:00` form; agents reading timestamps should accept either.

## HTML-Escaping Meta Values

Meta `content` attributes are attribute values, so HTML-escape anything the parser might misread:

| Raw | Escaped |
|-----|---------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `"` | `&quot;` |

```html
<!-- Title contains quotes -->
<meta name="pagium:title" content="The &quot;Big&quot; Idea">

<!-- Title contains ampersand -->
<meta name="pagium:title" content="Rust &amp; Tauri">
```

Plain single-quotes and non-ASCII characters do not need escaping; UTF-8 is the file encoding.

## Custom Meta Tags

Any `pagium:*` name beyond the core set is preserved. Use this for agent or plugin metadata:

```html
<meta name="pagium:custom-status" content="in-progress">
<meta name="pagium:agent-source" content="article-clipper">
<meta name="pagium:review-date" content="2026-05-01">
```

The convention `pagium:custom-<name>` is reserved for user metadata that was imported from a `custom` frontmatter field in markdown (see `pagium import`). If you are authoring a note directly, you can use any `pagium:*` name; just don't conflict with the core set.

## Round-Trip Guarantee

Parsing a note and re-serializing it preserves:

- Every meta tag name and value, byte-for-byte, **except** `pagium:modified`, which is refreshed on save.
- The order of meta tags in `<head>`.
- The body HTML exactly as written.
- All `<pagium-link>` elements and their attributes.
- Unknown elements and attributes — the parser does not strip anything.

If you edit a note by hand, the next `pagium save` will only rewrite `pagium:modified`. Your structure and custom tags survive.

## Bad Patterns to Avoid

- **Don't** put metadata in `<title>` or `<meta charset>` and expect Pagium to read it. The vault only inspects `pagium:*` names.
- **Don't** omit the `pagium:version` tag. Notes without it are rejected as malformed.
- **Don't** use relative timestamps ("yesterday", "2 hours ago"). The engine only parses ISO 8601.
- **Don't** mutate `pagium:created` after the note exists — agents may rely on it for sort order.
