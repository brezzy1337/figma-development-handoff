# figma-claude-code-handoff

A producer/consumer pair of Claude Code skills for **Figma developer handoffs**. Together they take a set of Figma screens, turn them into a labeled, routed handoff page an engineer can build from, then scaffold that handoff into application code.

## The problem it solves

A Figma file shows what screens look like — but not what to build, in what order, how the screens route to each other, or which states each one needs. These skills add that missing layer: a handoff that's legible to engineers *and* agents, and then a way to consume it into routes, components, and states.

## The two skills

| Skill | What it does |
|-------|--------------|
| **`figma-handoff-builder`** *(producer)* | Takes your existing Figma screens and builds a dedicated handoff page. Each screen is cloned and labeled with a stable `FLOW-## · Name` ID, tagged with a per-screen chip (Entry / Guard / Next / Route), wired to other screens with trigger-labeled connectors, and has its buttons renamed to the screen they navigate to (`PLAN-03 button`). It adds an `APP ENTRY` node for launch/redirect logic and writes a decoder legend to `.claude/figma-handoff-legend.md`. |
| **`figma-handoff-to-code`** *(consumer)* | Reads a handoff — the emitted legend plus the Figma file, or an export — and scaffolds the implementation: a route map, one screen component per `FLOW-##` ID (with its `.load`/`.empty`/`.err` states), navigation wiring, route guards, and launch/redirect logic — in whatever framework your repo already uses. It honors the decisions baked into the handoff instead of re-inventing the app's structure. |

## Install

```shell
/plugin marketplace add brezzy1337/figma-development-handoff
/plugin install figma-claude-code-handoff@figma-claude-code-handoff
```

Once installed, the skills are namespaced `figma-claude-code-handoff:figma-handoff-builder` and `figma-claude-code-handoff:figma-handoff-to-code`, and Claude reaches for them automatically when your request matches (a "dev handoff," a "routing map," "build this from the Figma flow," etc.).

> Requires the **Figma MCP** (`use_figma`, `get_metadata`, `get_screenshot`) for the producer side.

## How it works — the loop

1. **Design side:** `figma-handoff-builder` builds the handoff page from your Figma screens and writes `.claude/figma-handoff-legend.md`.
2. **Code side:** `figma-handoff-to-code` reads that legend (installing it from its own `assets/` on first run if absent) and scaffolds routes, components, and states.

A handoff produced this way is self-describing — the legend travels with it, so any reader, agent or human, can decode the conventions.

## What's inside

```
figma-claude-code-handoff/              # repo root = marketplace AND plugin
├── .claude-plugin/
│   ├── marketplace.json                # catalog; the one plugin entry has source "./"
│   └── plugin.json                     # plugin manifest
├── LICENSE
├── README.md
└── skills/
    ├── figma-handoff-builder/          # producer — builds the handoff page (+ emits the legend)
    │   ├── SKILL.md
    │   ├── assets/                     # figma-handoff-legend.md (emitted into the user's repo)
    │   └── references/                 # figma-mcp.md (safe-mutation guidance)
    └── figma-handoff-to-code/          # consumer — scaffolds code from a handoff
        ├── SKILL.md
        └── assets/                     # the legend + a CLAUDE.md pointer snippet
```

- **`assets/`** holds artifacts a skill *emits into your project*: `figma-handoff-legend.md` (written to `.claude/`) and `figma-handoff-to-code-claude-md.md` (a `CLAUDE.md` pointer you can paste in).
- **`references/`** holds guidance a skill *consults* but never installs: `figma-mcp.md` (safe-mutation rules for writing to Figma).

## Recommended CLAUDE.md pointer

Paste [`skills/figma-handoff-to-code/assets/figma-handoff-to-code-claude-md.md`](./skills/figma-handoff-to-code/assets/figma-handoff-to-code-claude-md.md) into your project's `CLAUDE.md` so Claude Code reaches for `figma-handoff-to-code` / `.claude/figma-handoff-legend.md` when implementing from a handoff.

## Validate

```bash
claude plugin validate .     # checks marketplace.json, plugin.json, and the skills' frontmatter
```

## License

MIT — see [LICENSE](./LICENSE).
