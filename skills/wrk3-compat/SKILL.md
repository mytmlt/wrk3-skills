---
name: wrk3-compat
description: Read-only compatibility check — analyze a project's docker compose files and local setup to decide if it can run under wrk3 parallel worktrees. Use before authoring any wrk3.yaml, when evaluating a new repo, or when `up` fails with port/name conflicts.
wrk3-version: v0.1.0
---

# wrk3 Compat Check

Goal: a **verdict** — `COMPATIBLE`, `COMPATIBLE WITH CHANGES`, or
`INCOMPATIBLE` — with the exact blockers and the minimal changes needed.
Read-only: inspect files, never boot containers, never edit the repo.

Rule 0: `wrk3.yaml` is executable configuration — never invent setup
steps. Every claim in the verdict must cite the file + line that proves it
(e.g. `docker-compose.yml:12`, `Makefile:8`, `README.md:20`).

## 1. Collect the facts

Run these in order; stop and report if the repo root can't be found.

1. **Repo root + git state.**
   `git rev-parse --show-toplevel` (config must live in the repo root).
   Note uncommitted compose changes — they affect the verdict.
2. **Compose candidates.** `ls docker-compose*.yml docker-compose*.yaml compose*.yaml compose*.yml`
   Read every file found. For each service record:
   - `image:` / `build:` (what it is)
   - `ports:` host bindings — verbatim strings (e.g. `"${APP_PORT:-4000}:3000"` vs `"4000:3000"`)
   - `container_name:` — exact value (any non-empty value is a blocker; wrk3's
     `up` fails fast on it because the name is global on the daemon and
     bypasses `compose -p <prefix>-<slug>` isolation)
   - `environment:` entries that consume `${VAR}` / `${VAR:-default}`
   - `volumes:` — distinguish named volumes (safe: isolated per compose
     project `<prefix>-<slug>`) from **bind mounts**: relative `./x:/y`
     (worktree-safe, each worktree gets its own copy) vs absolute
     `/data:/y` or `~/x:/y` (shared across worktrees → data corruption risk)
   - `network_mode: host` / `privileged: true` / `ports` with host IP pinning —
     flag each, they break isolation
   - `extends:` / multiple `-f` files / `profiles:` — note composition order
3. **Local setup lifecycle.** Read `Makefile`, `package.json` scripts,
   `README.md`, `CONTRIBUTING.md`, `Dockerfile(s)`, `.env.example`, migration
   dirs (`db/`, `migrations/`, `drizzle/`, `prisma/`). Record the canonical:
   - boot deps/infra (usually `docker compose up --wait --build`)
   - first-time setup (install, codegen, migrations, seed — exact commands)
   - foreground dev command (what `entry.run` should be)
   - stop / logs commands
   If the setup needs host-global state (single system DB, fixed socket path,
   license server on a hardcoded port, hardware device), record it — it may
   be incompatible.
4. **Host ports the stack binds.** Every host-side port that must differ per
   worktree. For each, note whether the compose file reads it from an env var
   (`${APP_PORT:-4000}` → parameterizable, good) or hardcodes it (`4000:3000`
   → conflict across worktrees until parameterized).

If anything is ambiguous (which compose file is canonical, what the real
setup sequence is), ask — do not guess.

## 2. Evaluate against the wrk3 isolation model

wrk3 gives each worktree: a git worktree dir, `allocated = base + index*step`
ports injected as env (`<NAME>_PORT`, uppercased), and a compose project
`-p <prefix>-<slug>` (volumes/networks/container names derived from it).
A project works with wrk3 iff all of these hold (or can be made to hold with
listed changes):

| # | Check | How | Blocker if |
| - | ----- | --- | ---------- |
| C1 | No `container_name:` | grep each compose file | any non-empty `container_name:` → `COMPATIBLE WITH CHANGES` (delete it; compose generates `<project>-<service>-1`) |
| C2 | Host ports parameterized | every `ports:` host side uses `${VAR:-default}` | hardcoded `"HOST:CTR"` → `COMPATIBLE WITH CHANGES` (parameterize to `"${X_PORT:-HOST}:CTR"`) |
| C3 | No `network_mode: host` | grep compose files | present → usually `INCOMPATIBLE` (ports can't be isolated per worktree) |
| C4 | No shared absolute bind mounts | inspect `volumes:` | absolute host path shared by all worktrees holding mutable state → `COMPATIBLE WITH CHANGES` (convert to named volume or relative path) or `INCOMPATIBLE` if the path is mandated by the toolchain |
| C5 | Setup commands are repo verbs | setup comes from Makefile/README/compose, runs via `sh -c` with `cwd=worktree`, `env=ports` | setup requires manual GUI steps, host-global daemons, or secrets you can't reproduce per worktree → `INCOMPATIBLE` or scoped `COMPATIBLE WITH CHANGES` |
| C6 | Ports fit `base + index*step` | count distinct host ports, check `ports.base` offsets don't collide with other wrk3 projects on the same host | two configs sharing a host with overlapping bases → `COMPATIBLE WITH CHANGES` (shift `ports.base` / `step`) |
| C7 | Heavyweight sanity | note image sizes, `setup` time, ports count | >2 min boot or >5 host ports → still compatible, but warn and require user approval before live `up` |

Named volumes, per-service `networks:`, `depends_on:`, and `healthcheck:`
are fine — they inherit `-p <prefix>-<slug>` isolation automatically.

## 3. Emit the verdict

Use this shape (copy into chat or a scratch file — never commit it):

```markdown
## wrk3 compat verdict: <COMPATIBLE | COMPATIBLE WITH CHANGES | INCOMPATIBLE>

**Repo root:** <path> | **Compose files:** <list> | **Services:** <names>

| Check | Result | Evidence |
| ----- | ------ | -------- |
| C1 container_name | pass/fail | <file:line> |
| C2 host ports parameterized | pass/fail | <file:line> |
| C3 host network mode | pass/fail | <file:line> |
| C4 bind mounts | pass/fail | <file:line> |
| C5 setup reproducible | pass/fail | <file:line> |
| C6 port range | pass/fail | <bases> |

**Required changes (if any):**
1. <file:line — exact edit, e.g. `delete container_name: testapp-db`>
2. <e.g. `"4000:3000"` → `"${APP_PORT:-4000}:3000"` + `ports.base: {app: 4000}`>

**Proposed ports.base mapping:**
| wrk3 name | .env var | compose host binding | default |
| --------- | -------- | -------------------- | ------- |
| app | APP_PORT | <service ports entry> | <n> |

**Suggested entry commands (repo verbs only):**
setup: [...] | run: "..." | stop: "..." | logs: "..."
```

Verdict rules:

- All C1–C6 pass → `COMPATIBLE` — hand off to `wrk3-setup` to author the config.
- Any fail fixable by editing compose/env/ports (C1, C2, C4-path, C6) →
  `COMPATIBLE WITH CHANGES` — list every edit with file:line, then hand off.
  Do not apply the edits without the user approving.
- C3 present, or C5 requires unreproducible host-global state →
  `INCOMPATIBLE` — say why, name the tool constraint, suggest the closest
  alternative (e.g. run that one service outside wrk3, or single-worktree mode).

Never run `add`/`up` (they create worktrees/containers). The handoff is the
verdict + evidence, not a config.
