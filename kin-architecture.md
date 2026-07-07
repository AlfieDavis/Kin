# Kin — Architecture & Build Spec

A handoff document for building the real Kin backend. The prototype (`kin.html`) demonstrates the model and every flow with in-memory data; this spec turns it into a persistent, multi-tenant, multi-family platform. It is written to be fed to Claude Code as the source of truth.

---

## 1. What Kin is

A genealogy platform where any family can keep its own tree. It launches with two seed families — **Blaker** (paternal) and **Rainback** (maternal) — but the whole point is that the same structure serves any family: people browse a public directory, open public families, request access to private ones, and request that a brand-new family line be created. Members add relatives; admins approve. Trees can be public or password-protected.

The one idea everything hangs on: **genealogy is a graph, not a tree.** Model it as a graph from day one or the "spouse who wants to add their own parents" case will be impossible to represent later.

---

## 2. The relationship model (read this first)

Two node types, one edge type. Do not model "parent" as a single pointer on a person, and do not store a spouse as a name string.

- **person** — a human. Carries their own identity and, crucially, one nullable pointer: `child_of_union_id`.
- **union** — a marriage or partnership joining two persons (`partner_a`, `partner_b`). Either partner may be null (single-parent lines are legal).
- **child → union** — a person descends from a *union*, not from a person. `person.child_of_union_id` points at the union they were born into.

That's it. Everything else is a query:

- **Children of X** = persons whose `child_of_union_id` is any union X partners in.
- **Parents of X** = the two partners of `X.child_of_union_id`.
- **Spouse of X** = the other partner in a union X belongs to.

### Why this answers the hard question

> "Someone starts a spouse's line and wants to add *their* parents."

Anne marries David. Anne is a full `person` (she can even belong to a different `family` — she does, in the seed data: she's Rainback, married into a Blaker union). To give Anne her own ancestry you don't touch David or the Blaker tree at all:

1. Create a new `union` (Anne's parents), e.g. `partner_a = Peter`, `partner_b = Joan`.
2. Set `anne.child_of_union_id = that union`.

Anne's line now extends upward independently, and either of *her* parents can be extended the same way, recursively, forever. The ancestor **fan chart** in the prototype walks exactly this: `child_of_union → partners → their child_of_union → …`. Empty wedges are unions/partners that don't exist yet, and clicking one runs the two-step above.

Cross-family membership (Anne is `family_id = rainback` but partners in a Blaker union) is a feature, not a bug — it's how two family trees legitimately share people. Keep `person.family_id` as the *primary* family for directory/permissions, and let unions cross families freely.

Multiple marriages: a person can partner in more than one union. The prototype renders only a "primary" union for layout simplicity; the schema below fully supports many.

---

## 3. Recommended stack

Chosen to sit comfortably on a self-hosted box behind Caddy with PM2, no managed services required.

| Layer | Choice | Notes |
|---|---|---|
| Runtime | Node 20+ | PM2-managed process |
| API | Fastify (or Express) | REST, JSON |
| DB | PostgreSQL 16 | recursive CTEs do the ancestor/descendant walks natively |
| ORM/migrations | Prisma or Drizzle | Drizzle if you want raw SQL closeness |
| Auth | Lucia or hand-rolled sessions (argon2 + httpOnly cookie) | JWT only if you need statelessness |
| Frontend | keep the current single-page app, or port to Vite/React | D3 charts already written |
| Proxy | Caddy | TLS + reverse proxy to the Node port |
| Files (photos) | local disk or S3-compatible (e.g. Hetzner Object Storage) | store keys, not blobs, in Postgres |

Postgres is the deliberate pick: ancestor and descendant traversals are `WITH RECURSIVE` queries, so the graph walks the prototype does in JS become one indexed query each.

---

## 4. Database schema

```sql
-- families -------------------------------------------------------
create table families (
  id            uuid primary key default gen_random_uuid(),
  slug          text unique not null,          -- 'blaker'
  name          text not null,                 -- 'Blaker'
  crest         text not null,                 -- 'B' (or an image key later)
  side          text,                          -- 'Paternal line'
  description   text,
  visibility    text not null default 'public' check (visibility in ('public','private')),
  password_hash text,                           -- null when public
  root_person_id  uuid,                          -- default descendant root
  pivot_person_id uuid,                          -- default ancestor-fan focus
  created_by    uuid references users(id),
  created_at    timestamptz default now()
);

-- persons (graph nodes) -----------------------------------------
create table persons (
  id            uuid primary key default gen_random_uuid(),
  family_id     uuid not null references families(id) on delete cascade,
  name          text not null,
  born          int,                            -- year; use a date col if you need days
  died          int,                            -- null = living
  gender        text,
  bio           text,
  photo_key     text,
  child_of_union_id uuid references unions(id) on delete set null,
  status        text not null default 'pending' check (status in ('approved','pending','rejected')),
  created_by    uuid references users(id),
  created_at    timestamptz default now()
);
create index on persons (family_id);
create index on persons (child_of_union_id);
create index on persons (status);

-- unions (graph edges: marriages/partnerships) ------------------
create table unions (
  id         uuid primary key default gen_random_uuid(),
  partner_a  uuid references persons(id) on delete set null,
  partner_b  uuid references persons(id) on delete set null,
  type       text not null default 'marriage' check (type in ('marriage','partnership')),
  started    int,
  ended      int,
  status     text not null default 'approved' check (status in ('approved','pending','rejected')),
  created_at timestamptz default now()
);
create index on unions (partner_a);
create index on unions (partner_b);

-- contacts & links (a person can have many) ---------------------
create table person_links (
  id uuid primary key default gen_random_uuid(),
  person_id uuid not null references persons(id) on delete cascade,
  kind text not null check (kind in ('link','contact')),  -- url vs email/phone
  value text not null
);

-- users & membership -------------------------------------------
create table users (
  id uuid primary key default gen_random_uuid(),
  email text unique not null,
  name text,
  password_hash text not null,
  created_at timestamptz default now()
);

create table memberships (
  user_id   uuid references users(id) on delete cascade,
  family_id uuid references families(id) on delete cascade,
  role      text not null check (role in ('owner','admin','editor','viewer')),
  primary key (user_id, family_id)
);

-- moderation queues --------------------------------------------
-- new persons/unions use status='pending' above. Edits to existing
-- records are proposed here so approval is auditable.
create table change_requests (
  id uuid primary key default gen_random_uuid(),
  family_id uuid references families(id) on delete cascade,
  person_id uuid references persons(id) on delete cascade,
  patch jsonb not null,                 -- fields being changed
  submitted_by uuid references users(id),
  status text not null default 'pending' check (status in ('pending','approved','rejected')),
  created_at timestamptz default now()
);

create table family_requests (           -- "start a new family line"
  id uuid primary key default gen_random_uuid(),
  name text not null,
  side text,
  visibility text default 'public',
  note text,
  requested_by uuid references users(id),
  status text not null default 'pending' check (status in ('pending','approved','rejected')),
  created_at timestamptz default now()
);

create table join_requests (             -- "let me into this private family"
  id uuid primary key default gen_random_uuid(),
  family_id uuid references families(id) on delete cascade,
  requested_by uuid references users(id),
  note text,
  status text not null default 'pending' check (status in ('pending','granted','denied')),
  created_at timestamptz default now()
);

create table audit_log (
  id uuid primary key default gen_random_uuid(),
  family_id uuid, actor uuid, action text, target text, meta jsonb,
  created_at timestamptz default now()
);
```

> The `persons ↔ unions` foreign keys are mutually referential; create both tables then add the `child_of_union_id` FK in a follow-up `alter table`, or defer constraints.

---

## 5. Roles & permissions

Per-family roles via `memberships`. A user can hold different roles in different families (in the seed data Jack is `owner` of Blaker but `viewer` of Rainback).

| Capability | owner | admin | editor | viewer |
|---|:--:|:--:|:--:|:--:|
| View approved records | ✓ | ✓ | ✓ | ✓ |
| Submit new kin (→ pending) | ✓ | ✓ | ✓ | ✓ |
| Approve/decline kin & edits | ✓ | ✓ | — | — |
| Edit records directly | ✓ | ✓ | ✓* | — |
| Grant/deny join requests | ✓ | ✓ | — | — |
| Manage member roles | ✓ | ✓** | — | — |
| Change visibility / password | ✓ | — | — | — |
| Delete the family | ✓ | — | — | — |

\* editors' edits can be direct or routed through `change_requests` — make it a per-family setting.
\** admins can manage editors/viewers but not other owners.

Private families: non-members get directory metadata only (name, member count, side) — never the records — until a `join_request` is granted or the password is entered. Public families are readable by anyone; adding still requires auth and still queues as pending.

---

## 6. Approval workflows

1. **New kin** — person created with `status='pending'`. Appears only to admins (in the console) until approved. On approve → `status='approved'`. On decline → `status='rejected'` (or hard delete). Any union created alongside inherits the same gate.
2. **Edits** — a `change_request` row with a `patch`. Admin approve applies the patch to the person and logs it.
3. **New family** — a `family_request`. Admin "Create family" provisions the `families` row + a seed root `person`, and grants the requester `owner`.
4. **Access to private family** — a `join_request`. Admin grant → insert `membership` (role `viewer` by default) + optionally reveal password.

Every approve/decline writes to `audit_log`.

---

## 7. API surface (REST)

```
Auth
  POST   /auth/register
  POST   /auth/login
  POST   /auth/logout
  GET    /auth/me

Families / directory
  GET    /families                       ?q=&visibility=   (public metadata)
  GET    /families/:slug                  (records if permitted)
  POST   /families                        (admin-provision; usually via request approval)
  PATCH  /families/:id                    (owner: visibility, password, root/pivot)
  POST   /family-requests                 (request a new line)
  GET    /family-requests                 (admin)
  POST   /family-requests/:id/approve|reject

Persons & unions
  GET    /families/:id/persons            ?status=approved
  POST   /families/:id/persons            body:{name,born,died,bio,relation,anchorId}
  PATCH  /persons/:id                     (direct edit or → change_request)
  DELETE /persons/:id
  POST   /persons/:id/approve|reject
  POST   /unions                          {partner_a,partner_b,type}
  PATCH  /unions/:id

Graph queries (server-side recursive CTEs)
  GET    /persons/:id/ancestors?generations=4
  GET    /persons/:id/descendants?depth=4
  GET    /persons/:id/relations           (parents, partners, children)
  GET    /families/:id/path?from=&to=      (relationship finder — BFS/CTE)

Access & membership
  POST   /families/:id/join-requests
  POST   /join-requests/:id/grant|deny
  GET    /families/:id/members
  PUT    /families/:id/members/:userId     {role}
  POST   /families/:id/invites             {email,role}
```

The `relation`/`anchorId` body on `POST persons` mirrors the prototype's add flow: `relation ∈ {child, partner, parent, root}`, and the server does the union wiring described in §2 so clients never hand-build unions for the common cases.

### Ancestor query (the fan chart), as a recursive CTE

```sql
WITH RECURSIVE up AS (
  SELECT p.id, p.name, p.born, p.died, 0 AS gen, p.child_of_union_id
  FROM persons p WHERE p.id = $1
  UNION ALL
  SELECT parent.id, parent.name, parent.born, parent.died, up.gen+1, parent.child_of_union_id
  FROM up
  JOIN unions u ON u.id = up.child_of_union_id
  JOIN persons parent ON parent.id IN (u.partner_a, u.partner_b)
  WHERE up.gen < $2
)
SELECT * FROM up;
```

Descendants is the mirror image (join persons whose `child_of_union_id` belongs to a union the current person partners in).

---

## 8. Auth & security notes

- Passwords (user and family) hashed with argon2id; family password verification gates record access server-side — never trust the client, unlike the prototype which checks in the browser for demo purposes.
- Sessions in httpOnly, SameSite=Lax cookies; CSRF token on mutations.
- Authorize every read/write against `memberships` + family `visibility`. A private family's `/persons` endpoint must 403 for non-members even if they guess the slug.
- Rate-limit `POST /persons` and request endpoints to blunt spam submissions.
- The prototype exposes demo family passwords on the gate screen — remove that in production.

---

## 9. What the prototype fakes (and where the seam is)

| Prototype (client, in-memory) | Production (server) |
|---|---|
| `DB.persons` / `DB.unions` objects | `persons` / `unions` tables |
| `unlocked` Set | verified `memberships` + granted join requests |
| Family password compared in JS | argon2 verify server-side, sets a scoped session |
| `state.admin` toggle | real role from `memberships` |
| `status:'pending'` on the object | same field, but enforced by API authorization |
| Ancestor walk in JS (`ancestorAt`) | recursive CTE in §7 |
| Submissions/requests as JS arrays | `change_requests` / `family_requests` / `join_requests` tables |

The data shapes are intentionally 1:1, so porting is "replace the in-memory maps with API calls," not a redesign.

---

## 10. Suggested build order for Claude Code

1. **Schema + migrations** (§4) and a seed script that loads Blaker + Rainback exactly as `kin.html` does (same names/dates makes visual diffing trivial).
2. **Auth** (register/login/session) and `memberships`.
3. **Read API**: families directory, family records, the two recursive graph queries. Point the existing frontend at these.
4. **Write API**: `POST persons` with `relation/anchorId` union-wiring, edits, unions.
5. **Moderation**: pending gate on reads for non-admins, approve/decline endpoints, `audit_log`.
6. **Requests**: family-requests and join-requests end to end.
7. **Roles console** wired to `memberships`; invites.
8. **Hardening**: authorization tests per role, rate limits, photo uploads, then deploy behind Caddy under PM2.

Ship 1–3 first; that alone makes the prototype real and browsable with live data. Everything after is additive.
