# Pagium Note Examples

Concrete, valid examples for common note shapes. Each example is ready to save to a `.html` file in a vault.

## Minimal Note

The smallest valid Pagium note:

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="Hello">
  <meta name="pagium:created" content="2026-04-18T12:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T12:00:00+00:00">
</head>
<body>
<p>Hello, Pagium.</p>
</body>
</html>
```

## Daily Note

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="2026-04-18">
  <meta name="pagium:created" content="2026-04-18T08:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T22:30:00+00:00">
  <meta name="pagium:tags" content="daily">
</head>
<body>
<h1>2026-04-18</h1>

<h2>Tasks</h2>
<ul>
  <li><input type="checkbox" checked> Review <pagium-link target="projects/alpha">Alpha</pagium-link> PR</li>
  <li><input type="checkbox"> Draft <pagium-link target="rfcs/plugin-system">plugin system RFC</pagium-link></li>
  <li><input type="checkbox"> Sync with <pagium-link target="people/jamie">Jamie</pagium-link></li>
</ul>

<h2>Notes</h2>
<p>Spent the morning on <pagium-link target="rust-notes#performance">Rust perf work</pagium-link>.
  The <code>scraper</code> crate is fast enough — no need for <code>html5ever</code> directly.</p>

<h2>Tomorrow</h2>
<p>Start on <pagium-link target="projects/beta">Beta</pagium-link> scoping.</p>
</body>
</html>
```

## Literature Note (Clipped Article)

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="Designing Data-Intensive Applications — Ch 1 Notes">
  <meta name="pagium:created" content="2026-04-18T14:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T15:20:00+00:00">
  <meta name="pagium:tags" content="books,distributed-systems">
  <meta name="pagium:custom-source" content="Kleppmann, 2017, Ch 1">
</head>
<body>
<h1>Reliability, Scalability, Maintainability</h1>

<blockquote>
  <p>Reliability means continuing to work correctly, even when things go wrong.</p>
</blockquote>

<h2>Key Concepts</h2>
<ul>
  <li><strong>Faults</strong> — one component deviating from spec. Not the same as failures.</li>
  <li><strong>Failures</strong> — the system as a whole stops providing the service.</li>
</ul>

<p>Compare with <pagium-link target="concepts/fault-tolerance">fault tolerance</pagium-link>
  — the two ideas are orthogonal but often conflated.</p>

<h2>My Take</h2>
<p>The chapter maps cleanly onto our work in <pagium-link target="projects/reliability-review">the reliability review</pagium-link>.</p>
</body>
</html>
```

## Code-Heavy Technical Note

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="SQLite WAL Mode Pragmas">
  <meta name="pagium:created" content="2026-04-18T09:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T09:30:00+00:00">
  <meta name="pagium:tags" content="sqlite,rust,concurrency">
</head>
<body>
<h1>Enabling WAL Mode</h1>
<p>For Pagium's concurrent GUI + CLI access, we open every connection with:</p>

<pre><code class="language-rust">conn.pragma_update(None, "journal_mode", "WAL")?;
conn.busy_timeout(Duration::from_millis(15_000))?;</code></pre>

<h2>Why 15 seconds?</h2>
<p>The GUI indexer can burst-write during a full reindex. 15s tolerates the worst case we've
  measured without cutting off agent workflows. See <pagium-link target="measurements/fts5-burst">burst measurements</pagium-link>.</p>

<h2>Trade-off</h2>
<p>WAL creates <code>.pagium/index.db-wal</code> and <code>.pagium/index.db-shm</code> sidecars.
  Add both to <code>.gitignore</code>.</p>
</body>
</html>
```

## Table-Driven Reference Note

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:version" content="1">
  <meta name="pagium:title" content="HTTP Status Codes">
  <meta name="pagium:created" content="2026-04-18T11:00:00+00:00">
  <meta name="pagium:modified" content="2026-04-18T11:00:00+00:00">
  <meta name="pagium:tags" content="reference,http">
</head>
<body>
<h1>HTTP Status Codes</h1>

<table>
  <thead>
    <tr><th>Code</th><th>Name</th><th>Notes</th></tr>
  </thead>
  <tbody>
    <tr><td>200</td><td>OK</td><td>Default success.</td></tr>
    <tr><td>201</td><td>Created</td><td>Include <code>Location</code> header.</td></tr>
    <tr><td>409</td><td>Conflict</td><td>See <pagium-link target="patterns/idempotency">idempotency patterns</pagium-link>.</td></tr>
  </tbody>
</table>
</body>
</html>
```

## Anti-Patterns (DO NOT DO)

### Missing required meta

```html
<!-- BAD: missing pagium:version and pagium:title -->
<!DOCTYPE html>
<html>
<head>
  <meta name="pagium:created" content="2026-04-18T12:00:00+00:00">
</head>
<body><p>...</p></body>
</html>
```
Will be rejected at save time.

### `<a>` for internal links

```html
<!-- BAD: internal link won't show up in backlinks -->
<p>See <a href="other-note.html">other note</a>.</p>

<!-- GOOD -->
<p>See <pagium-link target="other-note">other note</pagium-link>.</p>
```

### Including the extension in `target`

```html
<!-- BAD -->
<pagium-link target="my-note.html">...</pagium-link>

<!-- GOOD -->
<pagium-link target="my-note">...</pagium-link>
```

### Unescaped special characters in meta

```html
<!-- BAD: breaks the parser -->
<meta name="pagium:title" content="Rust & Tauri">

<!-- GOOD -->
<meta name="pagium:title" content="Rust &amp; Tauri">
```
