---
name: neo-skill
description: Use when working with the `neo` CLI (github.com/mamersfo/neo) — a Supabase-CLI-like
  tool for Neo4j that provisions a local Neo4j Community container via Docker and manages
  version-controlled Cypher migrations. Covers neo init, neo start, neo status, neo stop,
  neo migration new, neo db reset, neo db diff, the neo/neo.toml config file, seed.cypher,
  migration file conventions, _neo_migrations history, $NEO_LIB_DIR template overrides,
  and install/upgrade of the single static binary. Triggers on "neo init", "neo start",
  "neo db reset", "neo db diff", "neo migration", "Supabase CLI for Neo4j", or local Neo4j
  Docker dev workflows. Does NOT cover Cypher query authoring — use neo4j-cypher-skill.
  Does NOT cover production/Aura administration (backup, restore, users) — use
  neo4j-cli-tools-skill or neo4j-security-skill. Does NOT cover writing application
  driver code — use neo4j-driver-javascript-skill or neo4j-driver-go-skill.
allowed-tools: Bash
version: 1.0.0
---

# neo skill — Supabase-CLI-like tooling for Neo4j

`neo` is a single static Go binary that gives Neo4j the Supabase local-dev experience:
it provisions a local **Neo4j Community Edition** container via Docker and manages
**Cypher migrations from version-controlled files**. No Node/Python/Java runtime needed —
only a running Docker daemon (`DOCKER_HOST` is respected).

Source of truth: https://github.com/mamersfo/neo (`README.md`, `design.md` for ADRs).

## When to Use

- Setting up a throwaway/local Neo4j instance for development → `neo init` + `neo start`
- Managing schema/seed changes as versioned files → `neo migration new`, `neo db reset`
- Capturing the live schema back into a migration → `neo db diff`
- Getting local connection credentials/URLs → `neo status`
- Installing or upgrading the `neo` binary

## When NOT to Use

- **Writing/optimizing Cypher itself** → `neo4j-cypher-skill` (neo only *runs* the files)
- **Production, Aura, backup/restore, user management** → `neo4j-cli-tools-skill` / `neo4j-security-skill`
- **Full zero-to-app onboarding** (modeling + data + app) → `neo4j-getting-started-skill`
- **Bulk CSV/data import** → `neo4j-import-skill`

## Requirements

- Docker with a **running daemon** (`docker info` must succeed). Colima, Docker Desktop,
  OrbStack all work; `DOCKER_HOST` is honored.
- Nothing else — no language runtime.

## Install

Release tarball (recommended — bundles the `lib/` templates next to the binary):

```bash
# Resolve latest version + platform (asset names: neo_<ver>_<Os>_<arch>.tar.gz, e.g. neo_0.1.0_Darwin_arm64.tar.gz)
VERSION=$(curl -fsSL https://api.github.com/repos/mamersfo/neo/releases/latest \
  | grep '"tag_name"' | cut -d'"' -f4 | tr -d 'v')
OS=$(uname -s)                                  # Darwin | Linux
ARCH=$(uname -m); [ "$ARCH" = "aarch64" ] && ARCH=arm64   # x86_64 stays x86_64 on darwin assets
curl -fsSL "https://github.com/mamersfo/neo/releases/latest/download/neo_${VERSION}_${OS}_${ARCH}.tar.gz" \
  | tar xz
sudo mv neo /usr/local/bin/neo
sudo mv lib /usr/local/bin/lib                  # co-located templates (optional but recommended)
neo --version
```

Alternatives:

```bash
go install github.com/mamersfo/neo@latest       # needs Go toolchain
# or from source:
git clone https://github.com/mamersfo/neo && cd neo && make build   # → bin/neo
```

## Core workflow (the Supabase loop)

```bash
neo init                              # 1. create neo/neo.toml + neo/migrations/ + neo/seed.cypher
neo start                             # 2. pull neo4j:5.26, start container — prints password ONCE
neo status                            # 3. container state + bolt/http URLs (table hides password)
neo migration new "create person"     # 4. scaffold neo/migrations/<UTC-ts>_create_person.cypher
# ...edit the migration file...
neo db reset                          # 5. wipe + replay all migrations + run seed.cypher
neo db diff                           # 6. dump live schema (constraints+indexes) into a new migration
neo stop                              # 7. stop container, keep data (--remove to delete it)
```

## Command reference

| Command | Flags | Supabase equivalent | Notes |
|---|---|---|---|
| `neo init` | `--dir .` `--force` | `supabase init` | Local files only; **generates a random password** into `neo.toml`. No cloud sign-in. |
| `neo start` | `--timeout 60` `--pull missing\|always\|never` | `supabase start` | First run pulls the image (takes a while). Password printed **only here**. |
| `neo status` | `--output table\|json` | `supabase status` | `table` hides the password; `json` includes it (for scripts). |
| `neo stop` | `--timeout 10` `--remove` | `supabase stop` | Idempotent. `--remove` deletes the container (data volume goes with it). |
| `neo migration new <name>` | — | `supabase migration new` | Creates `neo/migrations/<UTC-timestamp>_<slug>.cypher`. |
| `neo db reset` | `--seed` `--timeout 120` | `supabase db reset` | Wipe → replay migrations in filename order → run `neo/seed.cypher` (skip with `--seed=false`). |
| `neo db diff` | `--output file` | `supabase db diff` | Schema-only dump into a new migration file. See limitations below. |

**Exit codes:** `0` ok · `1` usage/config error · `2` Docker error · `3` Neo4j/Cypher error.
Use these in scripts/CI to branch on failure type.

## Project layout (created by `neo init`)

```
neo/
  neo.toml            # config — commit everything EXCEPT the password
  migrations/         # versioned Cypher, applied in filename order
  seed.cypher         # demo/dev data, run at end of `neo db reset`
```

`neo/neo.toml`:

```toml
[neo]
image = "neo4j:5.26"   # LTS pin; set a 2025.x tag to test CalVer releases
bolt_port = 7687
http_port = 7474
user = "neo4j"
password = "<generated-on-init>"   # local secret — do not commit
```

Connect apps/drivers with `bolt://localhost:7687`, user/password from `neo status --output json`.

## Migration file rules (important — read before editing)

- **Multiple statements allowed** — split client-side; `;` inside strings/comments is respected.
- **Schema statements (`CREATE CONSTRAINT` / `CREATE INDEX`) must live in data-free files** —
  one statement runs per transaction. Don't mix DDL and data writes in one file.
- Applied history is recorded in `_neo_migrations` nodes (filename, checksum, applied_at).
  **Never edit an already-applied migration** — the checksum will no longer match; create a new one.
- Template scaffold has `-- up` section; `-- down` is optional and unused in v1 (reset replays up only).
- Keep migrations idempotent where possible (`MERGE`, `CREATE CONSTRAINT ... IF NOT EXISTS`).

## `neo db diff` limitations (v1, by design)

- **Schema only**: constraints + indexes. Graph data is never dumped.
- Ambiguous objects become commented `-- TODO(diff)` lines — review before committing.
- Lookup indexes are skipped (auto-managed); constraint-backed indexes point at
  `CREATE CONSTRAINT` instead of duplicating.
- No `--from/--to` state comparison (planned v2).
- **Always run `neo db reset` after a diff** to prove the generated file applies cleanly.

## `neo/lib` template resolution (ADR-002)

Templates (`neo.toml.tmpl`, `migration.tmpl.cypher`) resolve in this order:

1. `$NEO_LIB_DIR` env var (dev/operator override)
2. `<exe-dir>/lib` (e.g. `/usr/local/bin/lib` next to the binary)
3. Embedded defaults (compiled in — binary always works standalone)

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| exit code 2, "cannot connect to Docker" | Daemon not running — start Docker Desktop / `colima start`; check `DOCKER_HOST` |
| exit code 3 on `db reset` | Cypher error in a migration — fix the file, reset again (replay is from scratch, so editing unapplied files is safe) |
| port already in use | Another Neo4j on 7474/7687 — `neo stop --remove`, or change `bolt_port`/`http_port` in `neo.toml` |
| lost the password | It's in `neo/neo.toml` (and `neo status --output json`); `neo start` prints it only once |
| checksum mismatch | An applied migration was edited — restore it, put changes in a new migration |
| `command not found: neo` | Binary not on PATH — use `./neo` or move to `/usr/local/bin` |

## For agents: execution guidance

- Prefer running `neo` commands via Bash; never hand-roll `docker run neo4j` when the user
  asked for the `neo` workflow.
- After `neo db reset`, verify with `neo status` and a count query
  (via `neo4j-cypher-skill` tooling or the Query API) before claiming success.
- `neo status --output json` is the machine-readable path for extracting credentials into
  a `.env` (`NEO4J_URI=bolt://localhost:7687`, user/password from JSON).
- The tool targets **local dev only** — never point it at production/Aura.
