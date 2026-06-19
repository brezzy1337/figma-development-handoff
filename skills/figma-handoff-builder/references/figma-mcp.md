# Figma MCP: safe-mutation patterns

Read this before your first `use_figma` mutation. These are the hard-won rules that keep a handoff build from corrupting the user's file. They all serve one goal: **make every change small, inspectable, and impossible to silently destroy prior work.**

## Tool roles — keep them separate

| Tool | Role | Safe to fail? |
|------|------|---------------|
| `get_metadata` | Read the layer tree (structure, names, IDs, parentage, counts) | Read-only — safe |
| `get_screenshot` | Read the rendered appearance | Read-only — safe |
| `use_figma` | **Mutate** the document (clone, move, label, connect) | **No — a throw can revert the whole doc** |

Inspect with the first two. Mutate with the third. Never blur the line — don't write probing code inside `use_figma` to "find out" what's there; that's what `get_metadata` is for.

## Rule 0 — Load a page before you read or clear it

This Figma MCP uses **dynamic page loading**: a page's children only exist in memory once that page is loaded. Any page you did not *just create in the current call* — including a handoff page you built in a previous call — reads as **nearly empty** until you load it:

```js
const page = figma.root.children.find(p => p.id === HANDOFF_PAGE_ID);
if (!page) return "NOT_FOUND: page";
if (page.loadAsync) await page.loadAsync();   // without this, page.children may read as ~empty
```

Skipping this is silently catastrophic, in ways that never look like an error:

- A `page.children` walk returns 1 child instead of 150 — so a verification "passes" on a corrupt page, or your read disagrees with the render and `get_metadata` (which load the page their own way) and you waste an hour chasing a phantom "revert."
- A *clear-then-rebuild* step removes only the one visible child and then **stacks a second full copy** on top — instant overlapping-duplicate corruption of every screen.
- A find-and-rename can't see the target and silently **falls through to the original source frames** on another page, editing the user's working file.

If a read of a page you built earlier comes back implausibly small, the cause is almost always a missing `loadAsync` — not lost work. Related: you **cannot** assign `figma.currentPage = page` (it throws, and a throw can revert the doc); use `await figma.setCurrentPageAsync(page)` only if you truly must switch pages — usually you should just operate on the page by reference.

## Rule 1 — Never throw to read

A thrown, uncaught error inside `use_figma` doesn't just abort the current call. It can roll the **entire document** back to a stale snapshot, discarding work you completed earlier in the session. A single careless `node.children[0]` on a node that has no children can cost you the whole page.

Consequences:

- **Do all inspection with `get_metadata` / `get_screenshot`**, never by writing exploratory code that might throw inside `use_figma`.
- **Guard everything that could fail** and return a status string instead of letting it throw:

```js
// BAD — if the find fails, this throws and can revert the document
const frame = figma.currentPage.findOne(n => n.name === "ONB-01");
frame.x = 100; // throws if frame is null → catastrophe

// GOOD — guard, and always return normally
const frame = figma.currentPage.findOne(n => n.name === "ONB-01");
if (!frame) return "NOT_FOUND: ONB-01";
frame.x = 100;
return "OK: moved ONB-01";
```

- **End every `use_figma` call with a normal `return`.** A returned status (even "FAILED: …") is recoverable. An uncaught throw is not.

## Rule 2 — Verify structure with metadata, not just screenshots

A screenshot shows you pixels. It cannot show you that two frames are sitting in exactly the same place, perfectly overlapping. Duplicate or overlapping frames render as one clean image but **corrupt the layer tree** — and they're a common outcome of a clone/reposition that half-failed.

After any clone or layout batch, call `get_metadata` and check:

- The **frame count** matches what you intended to create.
- No two frames share identical coordinates unintentionally.
- Each clone has the **correct parent** (it's on the handoff page, not still on the source page).

Only trust a screenshot for *appearance* questions ("does the chip sit above the frame?"), never for *structure* questions ("did I create the right number of frames?").

## Rule 3 — Node IDs remap after clone/rebuild; re-query by name

When you clone or rebuild nodes, the IDs you captured a moment ago may no longer point where you think. Don't cache a node ID across a mutation and reuse it.

Pattern: **do all lookups by name, inside the same `use_figma` call that mutates**, and look everything up *before* you start mutating.

```js
// Resolve everything first, by name, in this call:
const page = figma.root.children.find(p => p.name === "🛠 Dev Handoff");
if (!page) return "NOT_FOUND: handoff page";
const src = figma.root.children.find(p => p.name === "Designs");
if (!src) return "NOT_FOUND: source page";

const targets = ["ONB-01", "ONB-02", "ONB-03"]
  .map(name => ({ name, node: src.findOne(n => n.name === name) }));
const missing = targets.filter(t => !t.node).map(t => t.name);
if (missing.length) return "MISSING: " + missing.join(", ");

// Only now mutate — IDs resolved above are valid within this call.
```

## Rule 4 — Cross-page clone pattern

To copy a frame from the source page onto the handoff page:

```js
const clone = sourceFrame.clone();      // 1. clone (lands on the same page as the source)
handoffPage.appendChild(clone);         // 2. move it onto the handoff page
clone.x = targetX; clone.y = targetY;   // 3. reposition into the layout
```

Do this per frame in a loop, accumulating a result string, and return the summary:

```js
const created = [];
for (const t of targets) {
  const clone = t.node.clone();
  handoffPage.appendChild(clone);
  clone.x = startX;
  clone.y = startY;
  startX += clone.width + GAP;          // left→right spine
  created.push(clone.name);
}
return "CLONED: " + created.join(", ");
```

## Rule 5 — Small, separately-verified passes

Don't build the whole handoff in one mega-call. One `use_figma` call per logical batch (e.g. "clone the onboarding flow", then separately "label the onboarding flow", then "connect it"). After each, inspect with metadata + screenshot before the next.

Why: if a pass fails, the damage is contained to that batch, and you can see exactly what state you're in before continuing. A 200-line single call that throws on line 180 can take everything with it.

## Rule 6 — Design files have no native connectors

`figma.createConnector()` is a **FigJam** API. In a `/design/` file it isn't available, so don't reach for it. Draw each connector by hand: a thin line (a 2px-tall rectangle, or `createLine`), a small arrowhead glyph (▶ horizontal, ▲ vertical) at the destination end, and a text label with the trigger word. Single-ended arrowheads are awkward on a plain `createLine` (its `strokeCap` applies to both ends), which is why a rectangle-plus-glyph is the reliable approach. Lean on layout to carry the obvious structure and reserve hand-drawn arrows for the transitions a reader can't infer from position.

## Rule 7 — Verify with a compact guarded read, not a full dump

After a clone/layout batch you must confirm structure, but `get_metadata` on a page of cloned screens returns an enormous tree. A cheaper, equally rigorous check is a **read-only `use_figma` pass** that walks `page.children` and returns one line per frame — name, rounded x/y/w/h, and a count of how many siblings share that name — then bails with that string. It surfaces the two things that matter (missing frames, overlapping/duplicate frames) in a few hundred bytes. This is reading-via-`use_figma`, which is fine *as long as it only reads and always returns* — the hazard the rules guard against is a throw, not the act of reading. For the same reason, `get_screenshot` with a guessed node id is harmless: it returns an error, never a document change.

## Quick pre-flight checklist before any `use_figma` mutation

- [ ] If I'm touching a page I built in an earlier call, did I `await page.loadAsync()` first?
- [ ] Am I only *reading*? → use `get_metadata` / `get_screenshot` instead.
- [ ] Have I resolved every node by name, up front, with null-guards?
- [ ] Does every failure path `return` a status string instead of throwing?
- [ ] Is this one small batch, not the whole app?
- [ ] For button renames: am I searching by visible text, scoped to the handoff page (not the source)?
- [ ] Do I have a metadata + screenshot verification planned for right after?
