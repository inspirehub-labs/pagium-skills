---
name: pagium-html
description: Create and edit Pagium notes. Pagium notes are plain HTML5 files with pagium:* meta tags in <head> and <pagium-link> custom elements for wiki-links. Use when working with .html files in a Pagium vault, or when the user mentions Pagium notes, pagium-link, pagium:title, or "HTML PKM".
---

# Pagium Note Format Skill

Create and edit valid Pagium notes. A Pagium note is a plain HTML5 document that opens correctly in any browser. Pagium adds two conventions on top of HTML: `pagium:*` meta tags in `<head>` for metadata, and `<pagium-link>` custom elements in `<body>` for wiki-links.

Standard HTML (headings, paragraphs, lists, code blocks, tables, images) is assumed knowledge. This skill only covers the Pagium-specific pieces.

## Workflow: Creating a Pagium Note

1. **Write the document frame** — `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`. All required.
2. **Add required meta tags** in `<head>`: `pagium:version`, `pagium:title`, `pagium:created`, `pagium:modified`. See [META.md](references/META.md) for all meta tags.
3. **Write the body** using ordinary HTML. Any valid HTML is allowed — lists, tables, code, images, whatever.
4. **Link between notes** with `<pagium-link target="slug">display</pagium-link>`. See [LINKS.md](references/LINKS.md) for resolution rules and broken-link behavior.
5. **Verify** — the file renders in a browser, and the Pagium parser round-trips it byte-for-byte (meta values preserved, body HTML untouched).

> Prefer letting the `pagium` CLI create and save files (`pagium create <slug> --title T`, `pagium save <slug>`). It guarantees the required meta tags, correct timestamps, and round-trip safety. Hand-written notes are for cases where the CLI is unavailable.

## Document Structure

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="Note Title">
  <meta name="pagium:created" content="2026-04-18T12:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T12:00:00+00:00">
  <meta name="pagium:tags" content="tag1,tag2">
</head>
<body>
  <!-- any valid HTML -->
</body>
</html>
```

## Required Meta Tags

| Tag | Required | Notes |
|-----|----------|-------|
| `pagium:version` | Yes | Format version. Currently `1`. |
| `pagium:title` | Yes | Human-readable title. HTML-escape special chars. |
| `pagium:created` | Yes | ISO 8601 UTC timestamp. |
| `pagium:modified` | Yes | ISO 8601 UTC timestamp. Updates on every save. |
| `pagium:tags` | No | Comma-separated. Example: `rust,pkm,active`. |

Additional `pagium:*` meta tags (e.g., `pagium:custom-field`) are allowed and preserved by the parser. Use them for agent-specific metadata; the core schema won't fight you.

See [META.md](references/META.md) for escaping rules, custom metadata conventions, and the round-trip guarantee.

## Wiki-Links

Links between notes use a custom element, not `<a href>`:

```html
<pagium-link target="other-note">another note</pagium-link>
```

- `target` is the slug (filename without `.html`).
- Subdirectories use forward slashes: `target="projects/foo"` resolves to `projects/foo.html`.
- Broken targets render with a visible broken-link indicator; they don't error.
- Use `<a href="https://...">` for external URLs only.

See [LINKS.md](references/LINKS.md) for slug normalization, subdirectory linking, and how the vault resolves link graphs.

## File Placement

- Extension: `.html`. Anything else is ignored by the vault indexer.
- Encoding: UTF-8.
- Location: anywhere under the vault root. Subdirectories are fine; the indexer walks recursively.
- Do **not** write into `.pagium/` — that's the engine's index directory.

## Rules (Round-Trip Guarantee)

Pagium's parser round-trips every note byte-for-byte. Keep this true:

1. `<!DOCTYPE html>` declaration required.
2. `<html>`, `<head>`, `<body>` tags required.
3. `pagium:version` and `pagium:title` required.
4. Escape HTML special characters in meta `content` attributes (`&lt;`, `&gt;`, `&amp;`, `&quot;`).
5. Preserve unknown elements and attributes — don't strip things you don't recognize.
6. `pagium:modified` is the only meta tag the engine will rewrite; everything else is yours.

Notes that violate rules 1–3 are rejected at save time.

## Complete Example

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="Project Alpha">
  <meta name="pagium:created" content="2026-04-18T10:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T12:30:00+00:00">
  <meta name="pagium:tags" content="project,active,rust">
  <meta name="pagium:custom-status" content="in-progress">
</head>
<body>
<h1>Project Alpha</h1>
<p>This project extends <pagium-link target="workflow-improvements">workflow improvements</pagium-link>
using <strong>Rust</strong> and <em>Tauri</em>.</p>

<h2>Tasks</h2>
<ul>
  <li>Initial planning <s>done</s></li>
  <li>Backend: see <pagium-link target="projects/alpha-backend">backend notes</pagium-link></li>
  <li>Frontend: see <pagium-link target="projects/alpha-frontend">frontend notes</pagium-link></li>
</ul>

<h2>Architecture</h2>
<pre><code class="language-rust">fn main() {
    println!("Hello, Pagium!");
}</code></pre>

<p>See also <pagium-link target="meeting-notes-2026-04-17">yesterday's meeting</pagium-link>.</p>
</body>
</html>
```

## When to Use the CLI Instead

If the `pagium` CLI is available in this environment, prefer it over hand-writing files:

```bash
# Create
echo '<p>hello</p>' | pagium create my-note --title "My Note"

# Read (returns parsed JSON)
pagium read my-note

# Save (body from stdin, preserves meta except modified)
pagium read my-note | transform | pagium save my-note
```

The CLI handles timestamps, meta defaults, slug resolution, and index updates. Load the [pagium-cli](../pagium-cli/SKILL.md) skill for the full command surface.

## References

- [Pagium landing page](https://pagium.app)
- [Note format v1 spec](https://github.com/inspirehub-labs/pagium/blob/main/spec/NOTE-FORMAT.md)
- [Round-trip guarantee](https://github.com/inspirehub-labs/pagium/blob/main/wikis/concepts/round-trip-guarantee.md)
