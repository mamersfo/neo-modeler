# neo-modeler

Local Neo4j dev environment built from scratch:

- [`neo`](https://github.com/mamersfo/neo) — Supabase-style CLI that provisions a
  `neo4j:5.26` Docker container and applies version-controlled Cypher migrations
  (`neo/migrations/`, replayed by `neo db reset`).
- The [Pi coding agent](https://github.com/earendil-works/pi-coding-agent) talks to the
  database through the official [neo4j MCP server](https://github.com/neo4j/mcp)
  (`.pi/mcp.json`).

## Use as template

This repo is a **GitHub template**. To create your own project from it:

1. Go to <https://github.com/mamersfo/neo-modeler>
2. Click **"Use this template"** → **"Create a new repository"**
3. Give your new repo any name you like (e.g. `my-project-neo4j`)
4. Clone your new repo and follow the Quickstart below

Your new repo will have the full migration structure, Pi MCP config, and agent
skills — ready to customize.

## Prerequisites

- **Docker** — `neo` provisions a local `neo4j:5.26` container.
- **[`neo` CLI](https://github.com/mamersfo/neo)** on `PATH` (`neo init`, `neo start`, `neo db reset`, …).
- **`jq`** — parses `neo status --output json` for MCP credentials and the download script.
- **`curl`** and **`tar`** — used by the `neo4j-mcp` download below.
- **[Pi](https://github.com/earendil-works/pi-coding-agent)** (optional) — only for the MCP agent workflow; `neo` and `cypher-shell` work without it.

## Quickstart

```bash
neo init      # one-time on a fresh clone: creates neo/neo.toml, neo/migrations/, neo/seed.cypher
             # (a local password is generated — see "Secrets" below)
neo start     # container up (first run pulls the image); password printed once
pi            # agent session — mcp__neo4j__get-schema / read-cypher / write-cypher
```

Migrations:

```bash
neo migration new "create schema"   # scaffold neo/migrations/<ts>_create_schema.cypher
# ... edit (schema-only files first, see AGENTS.md §4) ...
neo db reset                       # wipe + replay all migrations + seed.cypher
neo db diff                        # (optional) capture the live schema back into a migration
```

## Verify

After `neo start` (and once you've added at least one migration), confirm the database is
up and the schema applied:

```bash
neo status                        # container health + connection details
# in a Pi session, or via cypher-shell / any bolt client:
CALL dbms.components() YIELD name, versions, edition RETURN name, versions, edition;
MATCH (n) RETURN count(n) AS node_count;   # sanity check after migrations
```

If Pi lists the `neo4j` MCP server as `failed`, the container is down — run `neo start`
and reconnect via `/mcp` (see MCP wiring below).

## Secrets

`neo init` generates `neo/neo.toml` with a **local-only password**. That file is listed
in `.gitignore`, so it is never committed (per `AGENTS.md` §5 — never commit the
password). The version-controlled parts of `neo/` — `neo/migrations/` and
`neo/seed.cypher` — _are_ committed, so a fresh clone already has the schema and seed
and only needs `neo init` to (re)generate its local `neo.toml`. Recover the password
with `neo status --output json | jq -r '.password'`; `neo init` regenerates it if
missing. Don't hand-edit an already-applied migration — see `AGENTS.md` §4.

## MCP wiring

`.pi/mcp.json` runs the `neo4j-mcp` stdio server against the local database. The binary
at the repo root is **gitignored** — each machine downloads its own (below). Credentials
(URI, user, password) are resolved at server launch from `neo status --output json`
(`neo` and `jq` must be on `PATH`), so nothing secret is committed and the config
survives password changes. If Pi lists the server as `failed`, the container is down:
run `neo start` and reconnect via `/mcp`. Launch Pi from the repo root.

> **Safe to commit:** `.pi/mcp.json` contains no secrets. Its `NEO4J_MCP_*` values are
> derived at server launch from `neo status --output json`, and the DB password lives in the
> gitignored `neo/neo.toml` (see Secrets). Commit it freely.

### Download `neo4j-mcp`

Run from the repo root — `.pi/mcp.json` launches `./neo4j-mcp` from there. Requires
`curl`, `jq`, and `tar`. Resolves the latest release, verifies the checksum, and extracts
**only the binary** (the tarball also ships a `README.md` that would clobber this repo's):

```bash
VERSION=$(curl -fsSL https://api.github.com/repos/neo4j/mcp/releases/latest | jq -r '.tag_name | ltrimstr("v")')
OS=$(uname -s)                                          # Darwin | Linux
ARCH=$(uname -m); [ "$ARCH" = "aarch64" ] && ARCH=arm64; [ "$ARCH" = "x86_64" ] && ARCH=x86_64
curl -fsSLO "https://github.com/neo4j/mcp/releases/download/v${VERSION}/neo4j-mcp_${OS}_${ARCH}.tar.gz"
curl -fsSLO "https://github.com/neo4j/mcp/releases/download/v${VERSION}/neo4j-mcp_${VERSION}_checksums.txt"
shasum -a 256 -c --ignore-missing "neo4j-mcp_${VERSION}_checksums.txt"
tar xzf "neo4j-mcp_${OS}_${ARCH}.tar.gz" neo4j-mcp
rm "neo4j-mcp_${OS}_${ARCH}.tar.gz" "neo4j-mcp_${VERSION}_checksums.txt"
./neo4j-mcp --version
```

Notes:

- Linux: replace `shasum -a 256 -c --ignore-missing` with `sha256sum --ignore-missing -c`.
- Windows builds are `.zip` assets on the same releases page — extract and adjust the
  `command` in `.pi/mcp.json` (`neo4j-mcp.exe`).
- The binary arrives executable, and curl downloads don't trigger macOS Gatekeeper.
- To pin a specific version instead of the latest, set `VERSION=1.6.0` (say) and skip
  the GitHub API call.

See `AGENTS.md` for version-compatibility rules (this is Neo4j **5.26** — Cypher 5.x,
not 25), migration rules, and how the MCP tools fit the migration-first workflow.
