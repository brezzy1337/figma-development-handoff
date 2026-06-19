---
name: figma-development-handoff
description: Turn an existing set of Figma screens into a developer-facing handoff page — screens copied onto a dedicated page, labeled with a stable ID system (FLOW-## · Name), tagged with per-screen routing chips, and wired together with trigger-labeled connectors — so engineers know what to build, in what order, and how the screens relate. Use this skill whenever the user wants a "handover," "dev handoff," "developer handoff," a "screen flow" or "routing map," wants to show "how screens connect," or wants to prep / organize Figma designs for engineering — even if they don't say "skill" or name the exact format. Also trigger when someone has a set of Figma screens and asks what engineers should build first, how the screens route to each other, or how to organize designs for the eng team. Requires the Figma MCP (use_figma, get_metadata, get_screenshot).
---

# Figma Development Handoff

## What this produces

A single dedicated **handoff page** inside the user's Figma file that an engineer can read top to bottom and understand the app from. On that page:

- Every screen the design covers, **copied** (never moved — the originals stay put) onto the handoff page.
- Each screen carries a **stable ID** and a small **routing chip** that says where you arrive from, who's allowed in, and where you go next.
- Screens are **connected** with arrows labeled by the *trigger* — the action that causes the transition ("Tap Continue", "Submit", "Session expires").
- One **app-entry routing node** that captures the launch decision (e.g. returning user → dashboard, brand-new user → onboarding).
- Each **routing button renamed to the screen it leads to** (`PLAN-03 button`), so the destination is legible from the layer tree itself — which matters when a downstream agent like Claude Code reads the file rather than looking at it.
- A **legend emitted alongside the page** (a `.claude/figma-handoff-legend.md` and/or an on-canvas legend frame) so a downstream reader — the `figma-handoff-to-code` skill, or an engineer without it — can decode all of the above into routes, components, and states without guesswork.

The deliverable answers three engineering questions at a glance: *what do I build, in what order, and how do these screens relate?*

## How to work: the arc

This is collaborative and gated, not a fire-and-forget generator. The shape:

1. **Brainstorm the flow** with the user — what screens exist, how a user moves through them, what's onboarding vs. returning, what states each screen has.
2. **Propose the labeling system** for *their* app — the actual FLOW names, the IDs, the entry node. Show it as a plain list first.
3. **Get approval** on that naming before touching Figma. Renaming 30 frames after the fact is miserable; agreeing on the vocabulary up front is cheap.
4. **Copy screens** onto the handoff page.
5. **Label, name the routing buttons, and connect in small, separately-verified batches.**
6. **Screenshot + metadata check after each batch**, fix, continue.
7. **Done** — walk the user through the finished page.

Steps 4–6 happen through the Figma MCP, which has sharp edges that can destroy work if you're careless. **Read `references/figma-mcp.md` before your first `use_figma` mutation** — it covers the safe-mutation rules and the cross-page clone pattern. The single most important rule is summarized right below; do not skip it.

## ⚠️ The one rule that protects the user's work

**Never throw to read.** A thrown error inside `use_figma` does not simply roll back that one call — it can revert the whole document to a stale snapshot and wipe out work you already did this session. So:

- **Prefer `get_metadata` and `get_screenshot` for inspection** — they're read-only and can't corrupt anything. Reading *via* `use_figma` is fine only when the pass is guarded and always returns a string; never let a read throw. (And per Rule 0 in the reference, reading a page you built in an earlier call needs `await page.loadAsync()` first, or it reads as empty.)
- **End every `use_figma` call with a normal return.** No uncaught exceptions. If something might fail, guard it and return a status string instead of letting it throw.
- Work in **small passes you can verify independently**, so if one pass goes wrong the blast radius is one batch, not the whole page.

Everything else in the Figma-MCP reference exists to keep you on the safe side of this rule. Internalize it before you build.

---

## The labeling system (the core of this skill)

This vocabulary is what makes the handoff legible. Apply it consistently — its value is in being a *system*, not decoration.

### ID format: `FLOW-## · Name`

- **FLOW** — the area or context the screen lives in (e.g. `ONB` for onboarding, `DASH` for the dashboard, `BUDGET`, `AUTH`). Context, not feature.
- **##** — the screen's order *within that flow*, zero-padded (`01`, `02`, …). Ordering is the point: it tells an engineer the build/learn sequence.
- **Name** — a short human label.

Example: `ONB-03 · Set Budget`, `DASH-01 · Overview`, `AUTH-02 · Verify Email`.

### State suffixes

A screen often has more than one state. Mark them with a dotted suffix on the ID:

- `.load` — loading / skeleton
- `.empty` — empty state (no data yet)
- `.err` — error state
- **no suffix = the default / populated state**

So `DASH-01 · Overview`, `DASH-01.empty · Overview`, `DASH-01.load · Overview` are the same screen in three states — same number, different suffix. Engineers read this as "one route, multiple states to build."

Worked example: a "generate plan" step that designers drew as three separate frames — a loading screen, the populated result, and an error screen — is **one** screen at three states, not three screens. Label them `ONB-06 · Your AI Plan` (default), `ONB-06.load · Generating`, `ONB-06.err · Couldn't Generate`, stack the two states directly below the default, and draw the state transitions as short arrows ("plan ready" up from `.load`, "Try again" up from `.err`). This is usually the clearest place the suffix system earns its keep, since designers almost always draw states as peers.

### The per-screen chip

Above each frame, place a small text chip with **exactly these four lines**:

```
Entry:  where you arrive from
Guard:  who can reach it / what conditions must hold
Next:   where it routes to
Route:  the suggested in-app path (e.g. /onboarding/budget)
```

- **Entry** answers "how did the user get here?" — the inbound screen(s) or trigger.
- **Guard** answers "who's allowed here and when?" — auth state, feature flags, prerequisites ("must have completed ONB-02", "logged-in only").
- **Next** answers "where can they go from here?" — the outbound destination(s).
- **Route** is a suggested URL/route path, giving engineers a concrete handle.

The chip is the screen's passport. Keep the four lines even when one is "—"; the consistent shape is what makes the page scannable.

### Routing-button names

The chip and connectors describe routing for a *human* reading the page. But a downstream agent — Claude Code, say — mostly walks the **layer tree**, where a CTA is typically named something generic like "Frame" or "Button" and gives no hint of where it leads. So rename each routing button to encode its destination:

`<DEST-ID> button`  — e.g. `PLAN-03 button`, `ONB-06.load button`, `external button` (for an outbound link).

- Rename the button's **wrapping frame** (the tappable container). If the CTA is a bare text node with no wrapper, rename that text node instead — setting a text node's layer name doesn't change its visible text.
- Find buttons by their **stable visible text** ("Continue", "Build a new plan"), scoped to the handoff clone — never the original source frames, or you'll rename the user's working file.
- The same name recurring across screens is fine: `PLAN-01 button` can sit on three different screens, and on each it unambiguously means "this screen's button to PLAN-01." Within one screen each routing button points somewhere different, so names don't collide.

Now the routing is legible three ways — the chip's `Next`, the connector's trigger word, and the button's own `<DEST> button` name — so whoever picks up the handoff, human or agent, can follow it.

### The app-entry routing node

Add one distinct node (not a screen — a small decision node) representing **app launch**. It captures the branching that happens before any screen renders:

```
APP ENTRY
  onboarded?  → DASH-01
  new user?   → ONB-01
  logged out? → AUTH-01
```

This is where engineers learn the "first paint" logic, which otherwise hides between screens.

### The context rule (this one is subtle and important)

**The same task in a different context is a different screen.** Identity = task + context, not task alone.

The classic case: onboarding "Set Budget" (`ONB-03`, a full-screen step in a linear flow) is **not** the same screen as the returning user's "Edit Budget" (a modal opened from `DASH`). They may share fields, but their entry, guard, routing, and even layout differ. Give them separate IDs. Collapsing them hides real work and real routing from engineers.

When you're unsure whether two things are one screen or two, ask: *does a user reach them from different places, under different conditions?* If yes, they're two.

---

## Layout conventions

The page layout is itself information. Arrange so the structure is readable before anyone reads a single label.

- **Onboarding is a left→right spine.** Linear flows read as a horizontal sequence: `ONB-01 → ONB-02 → ONB-03 …`. The eye follows the arrow of time.
- **Dashboard tabs are a peer cluster.** Tabs are siblings, not a sequence — lay them out as a horizontal cluster of equals, not a chain.
- **Detail / child flows stack below their parent.** A screen you drill into from a parent sits *below* that parent, so vertical position encodes "deeper in."
- **Chips sit above their frame.** Always above, so the frame stays clean and the chips form a readable band.
- **Connectors carry the trigger word.** Every arrow is labeled with the action that causes the transition — the verb, from the user's point of view ("Tap Save", "Swipe to dismiss", "Token expires"). An unlabeled arrow is half a fact.

### Color: wayfinding vs. action

Read the palette off the design you're handing off, then hold one line: two roles, two colors, and don't cross them.

- **Active-tab / selected indicators → the UI's quiet "selected" color.** The indicator says "you are here." It's wayfinding, a statement of fact, not something to press.
- **The UI's accent color → reserved for action.** The accent means "do this." Putting it on a tab indicator would tell the engineer the indicator is interactive, which is a lie.

Keep the active-state indicator in the quiet/selected color so the accent always and only means "actionable." Use whatever those two colors actually are in the file — don't impose a palette of your own.

---

## The build process, in detail

Once the labeling is approved, build it on the Figma side. Do every Figma mutation through `use_figma`, and **read `references/figma-mcp.md` first** for the exact patterns. Two rules there will save you the most pain: never let a `use_figma` call throw (a throw can revert the whole document), and always `await page.loadAsync()` before reading or clearing a page you built in an earlier call — an unloaded page reads as nearly empty, which silently turns a "clear and rebuild" into "stack a second duplicate copy." The high-level sequence:

### 1. Create / locate the handoff page
Make a dedicated page (e.g. "🛠 Dev Handoff"). The originals are never touched — you clone *onto* this page.

### 2. Copy screens in a batch
Clone the source frames onto the handoff page using the cross-page clone pattern (clone → `appendChild` to the target page → reposition). **Rename each clone to its `FLOW-##` ID as you place it** (`c.name = 'ONB-02 · Budget'`) — that way the layer tree itself reads as the ID system, and later passes can re-find each frame by its stable ID name. After the batch, **verify with metadata, not just a screenshot** — perfectly-overlapping duplicate frames are invisible in a render but corrupt the layer tree. Metadata catches them; a screenshot won't.

Guard every pass for **idempotency**: before creating the page (or a batch of chips), check whether it already exists by name and bail with a status string if so. A re-run should never silently double the page or stack two chips on one frame.

### 3. Lay out the batch
Reposition the clones into the spine / cluster / stack conventions above.

### 4. Label the batch
Add the ID titles, the four-line chips, and the app-entry node.

### 5. Name the routing buttons
Rename each tappable that causes a screen transition to `<DEST-ID> button` (see *Routing-button names* above). Locate each by its **stable visible text** within the handoff clone, then rename the wrapping frame — or the text node itself for an unwrapped CTA. The buttons start out generically named ("Frame"), so search by text, not by name, and scope the search to the handoff page so you never touch the original source button.

### 6. Connect the batch
Draw the connectors and put the trigger word on each. **Design files have no native connector node** (that's a FigJam feature) — so in a `/design/` file you draw each connector yourself as a thin line (a 2px rectangle or a `createLine`), a small arrowhead glyph (▶ / ▲) at the destination end, and a text label carrying the trigger word. The layout already encodes most structure (a left→right spine reads as sequence, a cluster reads as peers), so connectors are most valuable for the *non-obvious* transitions: state loops, cross-flow jumps, and the app-entry branches. Don't crisscross the whole canvas — where two screens are far apart, the chip's Entry/Next lines already carry the routing, and a tangled arrow hurts more than it helps.

### 7. Verify the batch, then move on
Confirm structure (count of frames, no overlapping dupes, correct parentage) and that it *looks* right, then fix before continuing. `get_metadata` is the canonical read, but on a page full of cloned screens its full dump is huge; an efficient equivalent is a **guarded, read-only `use_figma` pass that returns a compact summary** — each top-level frame's name, x/y/w/h, and a per-name duplicate count — which catches missing frames and overlapping dupes without the token cost. (This is a read that happens to use `use_figma`; it's safe because it only reads and always returns a string — it never throws.) For appearance, `get_screenshot`; note that a wrong node id there just returns an error with zero document impact, so screenshot probing is harmless. Small verified passes mean any failure is contained to one batch.

Repeat 2–7 per flow (onboarding, then dashboard, then each detail flow). Don't try to do the whole app in one giant `use_figma` call — batch it.

### 8. Emit the handoff legend
A handoff is only useful if its reader knows how to decode it — so make the page self-describing. As a finishing step, emit the legend (`assets/figma-handoff-legend.md`, the same decoder the `figma-handoff-to-code` consumer skill ships): write it into the repo as **`.claude/figma-handoff-legend.md`** (where the consumer skill looks for it first), and/or drop a compact legend frame onto the Figma page itself. This lets a downstream agent (Claude Code with the consumer skill) or a human (without it) turn the page into routes, components, and states. If the project uses Claude Code, also offer a one-line pointer for its CLAUDE.md so the skill/legend is discoverable.

### 9. Hand it back
Walk the user through the finished page: the flows, the entry node, anything you flagged as ambiguous (e.g. screens you split under the context rule, or a transition whose trigger you guessed).

---

## A note on judgment

The labeling system is precise, but applying it to a real app takes interpretation — deciding flow boundaries, spotting where one "screen" is really two contexts, naming things an engineer will recognize. Lean on the brainstorm step to get this right *with* the user before you commit it to Figma, because the Figma side is the expensive part to redo.
