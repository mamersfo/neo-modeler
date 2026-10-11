# agents.md — Guidelines for AI agents building a Neo4j database from scratch

This repository bootstraps a local Neo4j database with the [`neo`](https://github.com/mamersfo/neo)
CLI (Supabase-style init/start/migrations/reset). Read this file before writing any Cypher,
config, or migration.

---

## 1. Version compatibility — read this first

**The `neo` tool pins `neo4j:5.26` by default in `neo/neo.toml`, while `neo4j-cypher-skill`
targets Neo4j 2025.x/2026.x (Cypher 25).**

These do **not** match. The local database you are building against is Neo4j **5.26**
(Cypher 5.x), not a 2025.x/2026.x release. Any Cypher produced under the assumption of
Cypher 25 will fail on this instance.

### Guideline: ask the Cypher skill for 5.x-compatible syntax

When invoking `neo4j-cypher-skill` (or any skill/prompt that writes Cypher), explicitly
instruct it:

> **"Produce Neo4j 5.x-compatible Cypher (target neo4j:5.26). No `CYPHER 25` version
> header, no Cypher 25 / GQL-conformance clauses (`LET`, `FINISH`, `FILTER`, `INSERT`),
> no 2025.x+ features (`CALL (x) {}`, `SHORTEST` QPE syntax, `SEARCH` clause, `GROUP BY`
> subclause, `cardinality()`, string interpolation, `UUID` type, map comprehension).
> Use the 5.x equivalents from the skill's Version Gates fallback column."**

Then **review the output against the trap table below** before running it — skills follow
their defaults unless told otherwise, and their default is `CYPHER 25`.

### Known 25 → 5.x differences to check

| Cypher 25 / 2025.x style (skill default) | Neo4j 5.26 equivalent |
|---|---|
| `CYPHER 25 <query>` | Omit the version header (or `CYPHER 15`) |
| `id(n)` deprecated → skill prefers `elementId(n)` | **`id(n)` is valid and preferred in 5.x.** `elementId(n)` also works in 5.x but returns the store element-id string (stable only within the same store) — pick deliberately and do not mix; if the skill "fixes" `id()` → `elementId()`, decide which semantics you want and apply it consistently |
| `CALL (x) { ... }` | `CALL { WITH x ... }` |
| `SHORTEST 1 (a)(()-[]->()){1,}(b)` | `shortestPath((a)-[*]->(b))` |
| QPE `(()-[:REL]->()){1,5}` | `[:REL*1..5]` |
| `SEARCH` (vector/fulltext) | `CALL db.index.vector.queryNodes(...)` / `CALL db.index.fulltext.queryNodes(...)` |
| `SET n:$($label)` / dynamic-label helpers in older forms | Verify against 5.26 docs; fall back to static labels if unsure |
| `LET` / `FINISH` / `FILTER` / `INSERT` (GQL clauses) | `WITH` / plain `RETURN`-less `WITH` / `WHERE` / `CREATE` |
| `GROUP BY` subclause, `cardinality()`, `uuid()` type, string interpolation `s"..."`, map comprehension | Skill's Version Gates fallback column (implicit grouping, `size()`/`length()`, `randomUUID()`, `+`/`toString()`, `apoc.map.*`) |
| `toIntegerOrNull` etc. | Fine — exists in 5.x; but `toInteger(null)` throws in 5.x too |

**Single source of truth for the version is `neo/neo.toml` → `[neo] image`.** If the
project decides to move to 2025.x/2026.x, change the pin there *first*, re-run
`neo db reset`, and only then relax this section. Never assume the version — confirm:

```bash
neo status
# then, e.g. via the HTTP query API or cypher-shell:
CALL dbms.components() YIELD name, versions, edition RETURN name, versions, edition;
```

---

## 2. Tooling: which skill for which job

| Task | Skill / tool |
|---|---|
| Provisioning the local DB, migrations, reset, status | `neo-skill` (`neo` CLI) — **this repo's default** |
| Writing/optimizing any Cypher | `neo4j-cypher-skill` — **but force 5.x output (§1)** |
| Graph data model design, labels/rels/constraints | `neo4j-modeling-skill` |
| CSV/JSON bulk import | `neo4j-import-skill` |
| Slow-query diagnosis (EXPLAIN/PROFILE) | `neo4j-query-tuning-skill` |
| Application driver code (JS/TS, Go) | `neo4j-driver-javascript-skill`, `neo4j-driver-go-skill` |
| Live DB inspection / ad-hoc queries from the agent | `neo4j` MCP server via `.pi/mcp.json` — see §6 |
| Production/Aura, backups, users | Not this repo — do not point `neo` at anything but localhost |

Do not hand-roll `docker run neo4j` when the `neo` workflow applies; `neo` owns the
container, password, and migration bookkeeping (`_neo_migrations` nodes).

---

## 3. Bootstrap workflow (from scratch)

```bash
neo init                        # creates neo/neo.toml, neo/migrations/, neo/seed.cypher
neo start                       # pulls neo4j:5.26, prints the password ONCE (it's in neo.toml)
neo status                      # verify container is up; --output json for credentials
neo migration new "create schema"   # scaffolds neo/migrations/<UTC-ts>_create_schema.cypher
# ... edit the migration (schema first, see §4) ...
neo db reset                    # wipe + replay all migrations + seed.cypher
neo db diff                     # capture live schema back into a migration (optional)
neo stop                        # stop container (--remove deletes data)
```

Typical from-scratch order:

1. **Model** the schema (labels, relationships, constraints) — `neo4j-modeling-skill`.
2. **Schema migration(s)** — constraints + indexes only, no data.
3. **Data migrations / seed** — `neo/seed.cypher` for demo data.
4. **Verify** — `neo status` + a count/version query before claiming success.

---

## 4. Migration rules (from neo-skill — violations break `db reset`)

- **Schema statements (`CREATE CONSTRAINT` / `CREATE INDEX`) must be in data-free files** —
  one statement runs per transaction. Never mix DDL and data writes in one file.
- **Never edit an already-applied migration** — the stored checksum in `_neo_migrations`
  will no longer match. Create a new migration instead. (Editing an *unapplied* file is
  safe — reset replays from scratch.)
- Keep migrations idempotent where possible: `MERGE`, `CREATE CONSTRAINT ... IF NOT EXISTS`.
- Use `//` for comments inside Cypher statements — `--` is SQL-style and rejected by
  Cypher (the scaffold's `-- up` section marker is template boilerplate; don't add
  `--`-style comments to your own statements).
- Applied order is filename order; `neo migration new` timestamp prefixes guarantee it.

---

## 5. Guardrails

- **Local only.** `neo` is a dev tool — never point it at production/Aura, and never
  commit the password from `neo/neo.toml` (commit everything else in `neo/`).
- **Confirm the running version before writing Cypher** (§1). If `neo.toml` says 5.26,
  write 5.26 Cypher.
- **Validate before executing:** `EXPLAIN` a write first; test reads with `LIMIT 1`;
  run `neo db reset` after schema changes to prove migrations apply cleanly.
- **Exit codes:** `0` ok · `1` usage/config · `2` Docker · `3` Neo4j/Cypher error —
  branch on these in scripts.
- If a generated query fails with a syntax error, suspect a **Cypher 25 feature leaking
  in from a skill default** before suspecting your data.
- **MCP writes are throwaway** — anything that must survive `neo db reset` belongs in a
  migration or `seed.cypher` (§6). Use `mcp__neo4j__write-cypher` only for experiments.

---

## 6. Pi MCP server (neo4j)

The Pi harness connects to the **official Neo4j MCP server** through the project config
`.pi/mcp.json` (stdio, `direct` exposure). Three tools:

- `mcp__neo4j__get-schema` — labels, relationship types, property keys (read-only)
- `mcp__neo4j__read-cypher` — read-only queries: verification, counts, `EXPLAIN`
- `mcp__neo4j__write-cypher` — arbitrary Cypher with write access

Wiring:

- **Binary:** `./neo4j-mcp` at the repo root (gitignored — drop the release binary there;
  `./neo4j-mcp --help` lists options). Requires `neo` and `jq` on `PATH`.
- **No credentials committed.** Env values in `.pi/mcp.json` are derived at server launch
  from `neo status --output json` (URI, user, password), so the config survives
  `neo init` re-runs and password changes. Never hardcode the password there.
- **The DB must be running.** If Pi lists the server as `failed`, the container is down:
  `neo start`, then reconnect via `/mcp` (a dropped connection retries on the next call).
- Launch Pi **from the repo root** — project MCP config is discovered from the session
  directory, not from subdirectories.
- Read/write is on by default. To restrict the agent to inspection, set
  `"NEO4J_MCP_READ_ONLY": "true"` in the `env` block of `.pi/mcp.json`.

**Division of labor:** migrations own the graph (§3–§4). Use the MCP tools for inspection,
verification, and throwaway experiments; durable schema/data changes go through
`neo migration new` + `neo db reset` — otherwise `neo db reset` wipes them and the model
drifts from the migration history. Cypher sent through MCP is subject to the same
**5.26 compatibility rules (§1)**.
