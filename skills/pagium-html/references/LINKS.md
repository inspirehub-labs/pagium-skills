# Wiki-Links Reference

Pagium uses a custom HTML element for internal links: `<pagium-link>`. This is the ONLY form Pagium recognizes for vault-internal links. Do not use `<a href="other-note.html">` — the link graph won't index it.

## Basic Syntax

```html
<pagium-link target="slug">display text</pagium-link>
```

- `target` — the slug of the target note (no `.html`, no leading `/`).
- Inner text — what the reader sees. Can be any inline HTML (bold, em, code).

## Examples

```html
<!-- Simple link -->
See <pagium-link target="intro">the intro</pagium-link>.

<!-- Inline formatting inside link text -->
<pagium-link target="rust-notes"><strong>Rust notes</strong></pagium-link>

<!-- Link inside a list -->
<ul>
  <li><pagium-link target="projects/alpha">Project Alpha</pagium-link></li>
  <li><pagium-link target="projects/beta">Project Beta</pagium-link></li>
</ul>

<!-- Link inside a code comment (still a real link) -->
<p>Also see <pagium-link target="design-decisions">design decisions</pagium-link>.</p>
```

## Slug Resolution

The `target` attribute resolves to a file in the vault:

| `target` | Resolves to |
|----------|-------------|
| `my-note` | `my-note.html` (vault root) |
| `projects/alpha` | `projects/alpha.html` |
| `meeting-notes-2026-04-18` | `meeting-notes-2026-04-18.html` |

Rules:

1. **No extension** in `target`. The engine appends `.html`.
2. **Forward slashes only** for subdirectories, even on Windows.
3. **No leading `/`.** Slugs are always vault-relative.
4. **Case-sensitive** on case-sensitive filesystems (Linux). Stick to one canonical casing.
5. **Spaces are allowed** (the indexer handles them) but prefer kebab-case for URL-safety.

## Broken Links

If `target` doesn't resolve to an existing file, the note still renders — the link is shown with a visible "broken" indicator in the editor. The engine tracks broken links; list them with:

```bash
pagium links broken
```

Broken links are not an error. They're a feature: you can link forward to notes you haven't written yet.

## Backlinks

Backlinks are derived from `<pagium-link target="...">` usage across the vault. Query them:

```bash
pagium links backlinks <slug>   # who links TO this note
pagium links outgoing <slug>    # who this note links TO
```

The parser only counts real `<pagium-link>` elements. HTML comments, text that looks like a link, and `<a>` tags do not contribute to backlinks.

## External Links

Use standard `<a href>` for external URLs:

```html
<p>Read the <a href="https://rust-lang.org">Rust homepage</a>.</p>
```

External links are NOT tracked in the link graph. They render normally in the browser and open in the system browser from the Pagium editor.

## Embeds / Transclusion

**Not supported in v1.** Obsidian's `![[embed]]` has no Pagium equivalent yet. To reference another note, use a regular `<pagium-link>`. Inline embedding is on the Phase 2 roadmap.

If you are importing a markdown vault with `pagium import`, `![[embed]]` syntax is converted to `<pagium-link>` with a warning by default, or aborts the import under `--fail-on-unsupported`.

## Common Mistakes

| Bad | Good | Why |
|-----|------|-----|
| `<pagium-link target="my-note.html">` | `<pagium-link target="my-note">` | Don't include the extension |
| `<pagium-link target="/my-note">` | `<pagium-link target="my-note">` | Don't use leading slash |
| `<pagium-link target="projects\alpha">` | `<pagium-link target="projects/alpha">` | Forward slashes only |
| `<a href="my-note.html">text</a>` | `<pagium-link target="my-note">text</pagium-link>` | `<a>` is not in the link graph |
| `<pagium-link target="my-note" />` | `<pagium-link target="my-note">my-note</pagium-link>` | Self-closing is fine in HTML5 but inner text is expected; prefer explicit text |
