# Handoff legend — reading a Figma developer-handoff page

This decodes a developer-handoff page produced by the **figma-handoff-builder** skill: a set of app screens copied onto one Figma page, each labeled and wired so you can implement the app from it — what to build, in what order, how screens relate, and where each control leads.

Read this first, then walk the page (via the Figma MCP `get_metadata`, or an export the producer emitted) and turn it into code. This legend is the contract between the design handoff and the implementation; honor it rather than re-deciding the app's structure.

## The vocabulary, and what each maps to in code

### Screen IDs — `FLOW-## · Name`
Every screen frame is named `FLOW-## · Name`.
- **FLOW** = the area / context (`ONB` onboarding, `DASH` dashboard, `AUTH`, `PLAN`, …). → group screens into feature modules/folders by FLOW.
- **##** = build / learn order *within that flow*, zero-padded. → implement ascending; `ONB-01` before `ONB-02`.
- **Name** = human label. → the screen/component name.

Each unique ID is **one route + one screen component**.

### State suffixes — `.load` / `.empty` / `.err`
An ID with a dotted suffix is a *state* of the base screen, not a separate screen:
- `.load` loading / skeleton · `.empty` no-data · `.err` error · **no suffix = default / populated**.

→ `ONB-06`, `ONB-06.load`, `ONB-06.err` are **one** route/component with three states to implement — not three routes.

### The chip (4 lines above each frame)
A text block above each screen with four fields:
- **Entry** — where the user arrives from (inbound screens / triggers). → the screens that navigate *here*.
- **Guard** — who may reach it / conditions (auth, feature flags, prerequisites). → route guards, redirects, conditional rendering.
- **Next** — outbound destinations. → the navigation targets *out* of this screen.
- **Route** — the suggested URL path (e.g. `/onboarding/budget`). → use as the route path.

A field shown as `—` means "none."

### Routing-button names — `<DEST-ID> button`
Interactive buttons are renamed to their destination: `PLAN-03 button`, `ONB-06.load button`, `external button`.
→ wire that button's press handler to navigate to the screen with that ID. `external button` = an outbound / third-party link, **not** an in-app route. The same `<DEST> button` name appearing on several screens just means each of those screens has its own button to that destination.

### APP ENTRY node
A distinct (non-screen) node listing launch branches, e.g. `new user → ONB-01`, `returning + active → DASH-01`, `returning + no data → DASH-01.empty`.
→ implement as the app's entry / redirect logic: which route renders first, by condition.

### Connectors — arrows with a word on them
Arrows between screens, each labeled with a **trigger** — the user action that causes the transition ("Continue", "Save", "Session expires").
→ the event that fires the navigation. Combine with `Next` and the `<DEST> button` names to build the full navigation graph.

### Layout = structure (read it before the labels)
- **Left→right row (a "spine")** = a linear flow, in order.
- **Horizontal cluster of peers** = sibling tabs (no order between them).
- **Stacked below a screen** = a detail / child screen reached from the one above.

### The context rule — don't merge
Two screens can share fields yet be different IDs (e.g. an onboarding "Set Budget" step vs. an "Edit Budget" modal). They differ in entry, guard, and route. → keep them as **separate** routes/components; do not collapse them into one because they look alike.

### Color (informational)
A quiet/selected color marks "you are here" indicators (active tab, current step) — non-interactive. The accent color marks actions (buttons). Use this only to tell interactive elements from wayfinding ones; it carries no routing meaning.

## How to consume it — procedure

1. **Inventory** every screen ID (with suffixes), each chip's four fields, every `<DEST> button`, the APP ENTRY branches, and the connectors (with their trigger words).
2. **Route map** — one route per *base* ID; path from the chip's `Route`.
3. **Build order** — ascending `##` within each flow; spine flows are sequential, clusters are independent, child screens come after their parent.
4. **State matrix** — for each ID, the default plus any `.load` / `.empty` / `.err` to implement.
5. **Navigation graph** — from the `<DEST> button` names + connectors (trigger = the event) + each chip's `Next`.
6. **Guards & entry** — chip `Guard` → route guards / conditional access; APP ENTRY → launch redirect logic.
7. **Scaffold**, honoring the context rule (separate IDs = separate components). Express routes, components, guards, and navigation in whatever framework the repo already uses — infer it, don't impose one.

**Confirm the route map + build order with the user before generating a lot of code.** Flag rather than guess: a `<DEST> button` whose target isn't a known ID, a chip field left `—` where you expected a value, a connector with no trigger word, or an `external` target with no URL.
