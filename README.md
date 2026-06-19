# figma-handoff

A Claude Code plugin bundling a producer/consumer pair of skills for **Figma developer handoffs** — build a labeled, routed handoff page from Figma screens, then scaffold code from it. This repo is both the plugin and the marketplace that distributes it.

## Install

```shell
/plugin marketplace add your-org/figma-handoff
/plugin install figma-handoff@figma-handoff
```

For local testing before pushing: `/plugin marketplace add ./figma-handoff` then the same install.

## What's inside

```
figma-handoff/                          # repo root = marketplace AND plugin
├── .claude-plugin/
│   ├── marketplace.json                # catalog; the one plugin entry has source "./"
│   └── plugin.json                     # plugin manifest
├── LICENSE
├── README.md
└── skills/
    ├── figma-development-handoff/      # producer — builds the handoff page (+ emits the legend)
    │   ├── SKILL.md
    │   ├── assets/                     # figma-handoff-legend.md (emitted into the user's repo)
    │   └── references/                 # figma-mcp.md (safe-mutation guidance)
    └── figma-handoff-to-code/          # consumer — scaffolds code from a handoff
        ├── SKILL.md
        └── assets/                     # the legend + a CLAUDE.md pointer snippet
```

## The two skills

| Skill | Role |
|-------|------|
| `figma-development-handoff` | **Producer.** Builds a dedicated handoff page from your Figma screens: stable `FLOW-## · Name` IDs, `.load`/`.empty`/`.err` state suffixes, a four-line chip per frame (Entry / Guard / Next / Route), routing buttons renamed to their destination (`PLAN-03 button`), an `APP ENTRY` decision node, and trigger-labeled connectors. Emits the decoder legend to `.claude/figma-handoff-legend.md`. Requires the Figma MCP. |
| `figma-handoff-to-code` | **Consumer.** Reads a handoff (the emitted legend + the Figma MCP, or an export) and scaffolds the implementation: a route map, one screen component per ID with its states, navigation wiring, route guards, and launch/redirect logic — in whatever framework the repo already uses. |

Installed, the skills are namespaced `figma-handoff:figma-development-handoff` and `figma-handoff:figma-handoff-to-code`, and Claude invokes them when your request matches.

### assets vs references

- **`assets/`** holds artifacts a skill *emits into your project*: `figma-handoff-legend.md` (written to `.claude/`) and `figma-handoff-to-code-claude-md.md` (a CLAUDE.md pointer you can paste in).
- **`references/`** holds guidance a skill *consults* but never installs: `figma-mcp.md` (the safe-mutation rules for writing to Figma).

## The loop

1. **Design side:** `figma-development-handoff` builds the handoff page and writes `.claude/figma-handoff-legend.md`.
2. **Code side:** `figma-handoff-to-code` reads that legend (installing it from its own `assets/` on first run if absent) and scaffolds routes, components, and states.

A handoff produced this way is self-describing: the legend travels with it, so any reader — agent or human — can decode the conventions.

## Recommended CLAUDE.md pointer

Paste [`skills/figma-handoff-to-code/assets/figma-handoff-to-code-claude-md.md`](./skills/figma-handoff-to-code/assets/figma-handoff-to-code-claude-md.md) into your project's `CLAUDE.md` so Claude Code reaches for `figma-handoff-to-code` / `.claude/figma-handoff-legend.md` when implementing from a handoff.

## Before you publish

Replace the placeholders `Your Name`, `you@example.com`, and `your-org` across `.claude-plugin/marketplace.json`, `.claude-plugin/plugin.json`, and `LICENSE`.

## Validate

```bash
claude plugin validate .     # checks marketplace.json, plugin.json, and the skills' frontmatter
```

## License

MIT — see [LICENSE](./LICENSE).
