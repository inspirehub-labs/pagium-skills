# Pagium Skills

Agent Skills for [Pagium](https://pagium.app), the HTML-native PKM desktop app.

Pagium stores every note as a plain `.html` file with `pagium:*` meta tags and `<pagium-link>` custom elements. These skills teach any skills-compatible agent how to author valid Pagium notes and drive the `pagium` CLI.

These skills follow the [Agent Skills specification](https://agentskills.io/specification) and work with Claude Code, Codex CLI, OpenCode, and any other skills-compatible agent.

## Installation

### Claude Code Marketplace

```
/plugin marketplace add inspirehub-labs/pagium-skills
/plugin install pagium@pagium-skills
```

### npx skills

```
npx skills add git@github.com:inspirehub-labs/pagium-skills.git
```

### Manually

#### Claude Code

Add the contents of this repo to a `/.claude` folder in the root of your Pagium vault (or whichever folder you're using with Claude Code). See [Claude Skills documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview).

#### Codex CLI

Copy the `skills/` directory into your Codex skills path (typically `~/.codex/skills`).

#### OpenCode

Clone this repo into the OpenCode skills directory:

```sh
git clone https://github.com/inspirehub-labs/pagium-skills.git ~/.opencode/skills/pagium-skills
```

Do not copy only the inner `skills/` folder — clone the full repo so the directory structure is `~/.opencode/skills/pagium-skills/skills/<skill-name>/SKILL.md`.

## Skills

| Skill | Description |
|-------|-------------|
| [pagium-html](skills/pagium-html) | Create and edit Pagium notes — HTML5 with `pagium:*` meta tags and `<pagium-link>` wiki-links |
| [pagium-cli](skills/pagium-cli) | Read, create, save, search, link-query, import, render, and watch a Pagium vault from the command line |

## Why a Skills Pack?

Pagium's native format is HTML. That's the whole point — HTML carries semantic structure, supports interactivity, and renders everywhere. But HTML is verbose, and agents without guidance tend to produce bloated, invalid, or inconsistent markup.

- **`pagium-html`** teaches the agent the minimum viable note shape: required meta tags, `<pagium-link>` rules, the round-trip guarantee. The agent emits clean files that Pagium indexes correctly.
- **`pagium-cli`** teaches the agent to operate the vault through the `pagium` binary — JSON/NDJSON output, safe concurrency with the GUI, `read | transform | save` pipelines, filesystem `watch` streams.

Together they turn a Pagium vault into an agent-native surface.

## Related

- [Pagium](https://github.com/inspirehub-labs/pagium) — the app
- [Pagium CLI design](https://github.com/inspirehub-labs/pagium/blob/main/docs/specs/2026-04-17-pagium-cli-design.md) — the authoritative spec for the command surface
- [Note format v1](https://github.com/inspirehub-labs/pagium/blob/main/spec/NOTE-FORMAT.md) — the authoritative file format spec
- [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills) — the Obsidian counterpart and the inspiration for this repo

## License

MIT — see [LICENSE](LICENSE).
