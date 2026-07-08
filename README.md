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
- **Timeline** — every dated life as a bar against a decade grid, with a
  "today" line; like the charts, it exports as a PNG.
- **How related?** — pick any two people and Kin traces the shortest path
  through the graph and names the relationship (siblings, spouses,
  2nd cousins once removed, great-aunt/uncle…).
- **Deep links** — `#/family/view/person` URLs; every record has a
  Copy Link action.
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

**The live site runs on a real backend now** — a REST API on the jackblaker.com
Cloudflare Worker with a D1 (SQLite) database, built from
[`kin-architecture.md`](kin-architecture.md):

- **Accounts**: email/password (PBKDF2, httpOnly session cookies) or
  **Sign in with Google** (Firebase ID tokens verified server-side).
- **Roles per family**: owner / admin / editor / viewer. Admins approve
  pending kin, proposed edits, and access requests — but only for the family
  they manage. One superadmin can do everything everywhere.
- **Moderation**: viewers' additions and edits queue as pending /
  change requests; email invites grant a role at first sign-in; every
  moderation action lands in an audit log.
- **Directory**: families are searchable and pinnable, so the switcher shows
  only what you care about.

The backend source lives in the site repo (`worker/kin/` — `api.js`,
`schema.sql`, `seed.sql`). This file (`kin.html`) is the whole frontend: it
boots against `/api/kin` when reachable and falls back to a self-contained
in-browser demo when opened from disk. [`kin-tokens.css`](kin-tokens.css) is
the design-token sheet.

Security notes for the prototype: passwords are compared as SHA-256 digests
client-side (demo-grade — the real gate belongs server-side, see the spec),
all rendered strings are escaped, links are restricted to `http(s)`/`mailto`,
and the page ships a strict Content-Security-Policy.
