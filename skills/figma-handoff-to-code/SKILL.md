---
name: figma-handoff-to-code
description: Interpret a developer-handoff page produced by the figma-development-handoff skill — its FLOW-## screen IDs, .load/.empty/.err state suffixes, per-screen Entry/Guard/Next/Route chips, destination-named buttons (e.g. "PLAN-03 button"), APP ENTRY node, and trigger-labeled connectors — and turn it into application code — a route map, one screen component per ID (with its loading/empty/error states), navigation wiring, route guards, and launch/redirect logic. Use this whenever the user wants to implement, scaffold, or build an app or feature from a Figma dev handoff, a screen-flow, or a routing map; references a figma-handoff-legend.md; or asks to turn FLOW-## screens or a handoff page into routes, screens, or components. Reads the handoff from the Figma MCP or an exported/emitted file. This is the consumer counterpart to the figma-development-handoff skill.
---

# Figma Handoff → Code

## What this is for

You are the **consumer** of a developer handoff. Someone (often the `figma-development-handoff` skill) produced a Figma page where every app screen is labeled with a stable ID, tagged with a routing chip, wired with connectors, and has its buttons named by destination. Your job is to translate that faithfully into code — a route map, screen components, their states, navigation, and guards — **without re-deciding the app's structure**. The handoff already made those decisions; honor them.

## Step 0 — Install the legend into `.claude/`, then read it

The conventions and what each maps to in code live in the **figma handoff legend**. Make it a persistent project artifact so every run — and any teammate — can use it:

1. Look for **`.claude/figma-handoff-legend.md`** in the project.
2. **If it's missing, create it:** write this skill's bundled `assets/figma-handoff-legend.md` to `.claude/figma-handoff-legend.md` (create the `.claude/` folder if it isn't there). If it already exists — e.g. the producer skill emitted it with this handoff — **leave it as-is**; it's authoritative for this project and may carry handoff-specific notes.
3. **Read `.claude/figma-handoff-legend.md`** before interpreting anything.

This self-installs the legend on first run, reuses it afterward, and lets someone open it without the skill installed.

## Step 1 — Get the handoff data

Two sources, in order of preference:

1. **An export the producer emitted** — a structured dump (JSON/markdown of the screens, chips, and buttons) sitting alongside the legend. Cheapest and most stable.
2. **The Figma file directly, via the Figma MCP.** Use `get_metadata` (read-only) to walk the page's layer tree:
   - screen **frame names** = the IDs (`FLOW-## · Name`, with any suffix),
   - **button names** = `<DEST> button` (the navigation targets),
   - the **chip** above each frame = four text lines (`Entry` / `Guard` / `Next` / `Route`),
   - the **APP ENTRY** node and the **connector** labels (the trigger words).
   `get_metadata` and `get_screenshot` are read-only and safe. If you ever need to read a Figma *page you didn't just open* via `use_figma`, that page may read as empty until you `await page.loadAsync()` — but for consuming a handoff you should rarely need to write to Figma at all.

If neither source is available, ask the user for the Figma file/page or an export rather than guessing the structure.

## Step 2 — Build the plan (and confirm it)

Derive these four artifacts from the handoff, then **show them to the user and get a go-ahead before scaffolding** — the same gated style the producer used, because generating a whole app's worth of code off a misread is expensive:

- **Route map** — one route per base ID, path taken from each chip's `Route`.
- **Build order** — ascending `##` within each flow; a left→right spine is sequential, a peer cluster is independent, child screens follow their parent.
- **State matrix** — for each ID, the default state plus any `.load` / `.empty` / `.err` to implement.
- **Navigation graph** — every edge from a `<DEST> button` (its press → navigate to that ID) and every connector (its trigger word → the event that fires it), cross-checked against each chip's `Next`.

## Step 3 — Scaffold

Implement against the plan, in the **framework the repo already uses** — infer it from the project (its router, component conventions, state patterns); do not impose a stack of your own. For each base ID: a screen component carrying its states, its route (from `Route`), its guard (from `Guard`), and its outbound navigation wired from the `<DEST> button` handlers. Implement the `APP ENTRY` branches as the launch/redirect logic.

## The conventions you must respect

- **Suffixes are states, not screens.** `ONB-06`, `ONB-06.load`, `ONB-06.err` = one component with three states. Don't make three routes.
- **The context rule.** Two similar-looking screens with different IDs are different components — different entry, guard, route. Never collapse them, even if they share fields.
- **`<DEST> button` = a navigation target.** Wire the handler to route to that ID. `external button` = an outbound link, not an in-app route.
- **Chip `Guard` = access control.** Turn it into route guards / redirects / conditional rendering, not just a comment.
- **Layout already told you the shape** before you read a single label — sequence, peers, or parent/child. Let it inform the module structure.

## Flag, don't guess

Surface ambiguities to the user instead of inventing answers: a `<DEST> button` whose target isn't a known ID; a chip field left `—` where a value was expected; a connector with no trigger word; an `external` target with no URL; or two IDs that look like they should be one (confirm the context rule was intended). A handoff is a spec, and a spec with a gap is worth a question.

## Relationship to the producer

This skill is the read/implement half of a pair. The write/design half is `figma-development-handoff`, which builds the handoff page and emits the legend. If you find yourself wanting to *change* the handoff (rename IDs, re-route screens), that's the producer's job — do it there so the page stays the single source of truth, then consume the updated version here.
