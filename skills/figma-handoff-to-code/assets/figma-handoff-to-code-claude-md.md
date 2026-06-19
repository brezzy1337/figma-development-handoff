<!-- Paste this section into your project's CLAUDE.md -->

## Figma developer handoffs

This project is implemented from **developer-handoff pages** produced by the `figma-development-handoff` skill. When asked to implement, scaffold, or build from a Figma handoff, screen-flow, or routing map:

- Use the **`figma-handoff-to-code`** skill — or read **`.claude/figma-handoff-legend.md`** — to interpret the conventions before writing code.
- Screen frames are named **`FLOW-## · Name`** → one route + one screen component each; implement in ascending `##` order within each flow.
- Suffixes **`.load` / `.empty` / `.err`** are *states* of that one screen, not separate routes.
- The **chip** above each frame gives **Entry / Guard / Next / Route** → inbound screens / access guards / outbound targets / the route path.
- Buttons named **`<DEST> button`** navigate to that screen ID (`external button` = an outbound link).
- The **`APP ENTRY`** node defines launch / redirect logic.
- **Context rule:** two distinct IDs are distinct components even if they look similar — never collapse them.

Confirm the route map and build order with the requester before generating a large amount of code.
