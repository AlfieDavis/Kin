# Kin — a family record as a graph

**Live: [jackblaker.com/kin](https://jackblaker.com/kin/)**

Kin is a genealogy app built on one idea: **a family is a graph, not a tree.**
Two node types — *persons* and *unions* (marriages/partnerships) — and one rule:
a child descends from a union, not from a person. That single choice means any
line can grow in any direction: a spouse who married in can add *their* parents,
grandparents, and whole ancestry without touching anyone else's branch.

![Kin](Kin.png)

## What it does

- **Descendant tree** — couples share a marriage bar, children hang below (D3).
- **Ancestor chart** — a horizontal pedigree; dashed plates are missing parents,
  click one to add that ancestor. Configurable 2–5 generations.
- **Records** — searchable card view with family statistics (living, unions,
  birth span, average lifespan).
- **How related?** — pick any two people and Kin traces the shortest path
  through the graph and names the relationship (siblings, spouses,
  2nd cousins once removed, great-aunt/uncle…).
- **Multi-family platform** — public and password-protected families, a
  directory, join requests, new-family requests, an admin console with an
  approval queue and per-family roles.
- **Persistence** — every change autosaves to the browser (localStorage).
  Export/import full JSON backups, or export **GEDCOM 5.5.1** for use in any
  other genealogy software.
- **Settings** — twelve live options: dark/light theme, four accent palettes,
  plate size, connector style, ancestor depth, date display, living markers,
  animations, autosave, confirmations, background glow.

## Running it

It's one file. Open `kin.html` in a browser (or serve the folder — the family
password gates use `crypto.subtle`, which needs HTTPS or localhost).

## Architecture

The prototype keeps everything in memory + localStorage, but the data shapes
are deliberately 1:1 with a real backend schema — persons, unions, families,
memberships, change requests. [`kin-architecture.md`](kin-architecture.md) is
the full build spec for the production version (Postgres recursive CTEs for the
graph walks, Fastify API, per-family roles). [`kin-tokens.css`](kin-tokens.css)
is the design-token sheet.

Security notes for the prototype: passwords are compared as SHA-256 digests
client-side (demo-grade — the real gate belongs server-side, see the spec),
all rendered strings are escaped, links are restricted to `http(s)`/`mailto`,
and the page ships a strict Content-Security-Policy.
