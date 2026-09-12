---
name: wrk3-setup
description: Set up wrk3 parallel worktrees for a repo — check compose/setup compatibility, propose minimal fixes when incompatible, then author and validate wrk3.yaml. Use when setting up wrk3, evaluating a repo, adding a stack, or fixing port/name conflicts or a broken config.
wrk3-version: v0.1.0
---

# wrk3 Setup

Goal: a working `wrk3.yaml` for the target repo, verified through
`fetch` + `status` (and `add`/`up` when the user approves running real
workloads). No registration step — wrk3 finds `wrk3.yaml`/`wrk3.yml`
walking up from cwd (override with `-f/--file <path>`).

If the repo is not compatible as-is, propose the minimal changes that
would make it compatible — then apply them (or work around them) only
with user approval before authoring the config.

Rule 0: `wrk3.yaml` is executable configuration — `entry.setup/run/stop`
run via `sh -c` on the user's host. Only write commands taken from the repo's
own docs/Makefile/compose files. Never invent setup steps. Every
compatibility claim must cite the file + line that proves it
(e.g. `docker-compose.yml:12`, `Makefile:8`, `README.md:20`).

## 0. Compatibility triage (read-only — run first, change nothing)

### 0a. Collect the facts

Stop and report if the repo root can't be found.

1. **Repo root + git state.** `git rev-parse --show-toplevel` (config must
   live in the repo root). Note uncommitted compose changes — they affect
   the verdict.
2. **Compose candidates.** `ls docker-compose*.yml docker-compose*.yaml compose*.yaml compose*.yml`.
   Read every file found. For each service record:
   - `image:` / `build:` (what it is)
   - `ports:` host bindings — verbatim strings (e.g. `"${APP_PORT:-4000}:3000"`
     vs `"4000:3000"`)
   - `container_name:` — exact value (any non-empty value is a blocker;
     wrk3's `up` fails fast on it because the name is global on the daemon
     and bypasses `compose -p <prefix>-<slug>` isolation)
   - `environment:` entries consuming `${VAR}` / `${VAR:-default}`
   - `volumes:` — named volumes (safe: isolated per compose project
     `<prefix>-<slug>`) vs **bind mounts**: relative `./x:/y`
     (worktree-safe, each worktree gets its own copy) vs absolute
     `/data:/y` or `~/x:/y` (shared across worktrees → corruption risk)
   - `network_mode: host` / `privileged: true` / `ports` with host IP
     pinning — flag each, they break isolation
   - `extends:` / multiple `-f` files / `profiles:` — note composition order
3. **Local setup lifecycle.** Read `Makefile`, `package.json` scripts,
   `README.md`, `CONTRIBUTING.md`, `Dockerfile(s)`, `.env.example`,
   migration dirs (`db/`, `migrations/`, `drizzle/`, `prisma/`). Record the
   canonical:
   - boot deps/infra (usually `docker compose up --wait --build`)
   - first-time setup (install, codegen, migrations, seed — exact commands)
   - foreground dev command (what `entry.run` should be)
   - stop / logs commands
   If setup needs host-global state (single system DB, fixed socket path,
   license server on a hardcoded port, hardware device), record it — it may
   be incompatible.
4. **Host ports the stack binds.** Every host-side port that must differ per
   worktree. For each, note whether compose reads it from an env var
   (`${APP_PORT:-4000}` → parameterizable, good) or hardcodes it
   (`4000:3000` → conflict across worktrees until parameterized).

If anything is ambiguous (which compose file is canonical, what the real
setup sequence is), ask — do not guess. Never run `add`/`up` during
triage (they create worktrees/containers).

### 0b. Evaluate against the wrk3 isolation model

wrk3 gives each worktree: a git worktree dir, `allocated = base + index*step`
ports injected as env (`<NAME>_PORT`, uppercased), and a compose project
`-p <prefix>-<slug>` (volumes/networks/container names derived from it).
A project works with wrk3 iff all of these hold (or can be made to hold
with listed changes):

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

### 0c. Emit the verdict

Use the template in `examples/verdict-template.md` (copy into chat or a
scratch file — never commit it):

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

- All C1–C6 pass → `COMPATIBLE` — continue to §1.
- Any fail fixable by editing compose/env/ports (C1, C2, C4-path, C6) →
  `COMPATIBLE WITH CHANGES` — continue to §1 (remediation), not §2.
  Do not apply edits without user approval.
- C3 present, or C5 requires unreproducible host-global state →
  `INCOMPATIBLE` — stop. Say why, name the tool constraint, suggest the
  closest alternative (e.g. run that one service outside wrk3, or
  single-worktree mode). Do not author a config.

## 1. Remediation (only for `COMPATIBLE WITH CHANGES` — needs approval)

1. List every required edit with file:line evidence (e.g. delete
   `container_name:`, `"4000:3000"` → `"${APP_PORT:-4000}:3000"`,
   absolute bind → named volume, shifted `ports.base`/`step`).
2. Get user approval before applying anything or working around it in the
   wrk3 config. Say which edits you will make vs. which conflicts the user
   accepts.
3. After approval, apply the edits (or the agreed workaround), then re-run
   the §0 checks on the changed files before continuing — only proceed to
   §2 with a `COMPATIBLE` (or approved `WITH CHANGES`) verdict.

## 2. Inspect the repo

Collect these facts before writing anything (reuse the §0 output —
don't re-read files you already cited):

1. **Repo root.** The config always lives in the repo root, so no repo
   path is stored — just place `wrk3.yaml` there. Get the root with
   `git rev-parse --show-toplevel` inside the checkout.
2. **Compose files.** Confirmed list from §0: service names, published
   host ports (`ports:`), env vars consumed (`${VAR:-default}`), named
   volumes, and networks.
3. **Dev lifecycle.** Confirmed canonical commands from §0:
   - boot deps/infra (e.g. `docker compose up --wait --build`)
   - first-time setup (install, codegen, migrations, seed)
   - foreground dev server (e.g. `npm run dev`, `docker compose logs -f`)
   - stop (e.g. `docker compose down`)
   - logs (e.g. `docker compose logs -f`)
4. **Host ports the stack binds.** Every host-side port that must differ per
   worktree needs a `ports.base` entry. Map each to the `.env` var the
   compose file actually reads (see port table below). Reuse the
   `ports.base` mapping proposed by the verdict.
5. **Branch names** the user wants in parallel (for slug/validation context).

If anything is ambiguous (which compose file is canonical, what the setup
sequence is, which ports matter), ask — do not guess entry commands.

## 3. Author `wrk3.yaml`

Start from the template in `examples/compose-stack.yaml` (this skill dir) or
the `wrk3` repo's `wrk3.yaml.example`. Write the new config to a **local,
gitignored** path in the repo root (`wrk3.<name>.local.yaml`) unless the
user explicitly wants a committed template — the config must live in the
repo root (its directory is the repo root).

```yaml
project:
  worktreeBase: .worktrees           # relative => <repoRoot>/.worktrees
source:
  type: git                          # only backend that ships in v1
  git: {remote: origin, fetchPrune: true}
runner:
  type: docker                       # portainer/nomad are stubs (not implemented)
  docker:
    composeFiles: [docker-compose.yml]  # must exist in the repo
    projectPrefix: demo                 # lowercase, short; compose -p <prefix>-<slug>
entry:
  setup: ["docker compose up --wait --build"]  # verbatim repo commands, in order
  run: "docker compose logs -f"
  stop: "docker compose down"
  logs: "docker compose logs -f"
ports:
  base: {app: 8000}                # every host port your stack binds (+ `app` required)
  step: 100                        # allocation = base + index*step
```

Constraints (enforced by `config.Validate` — unknown values fail fast):

- `source.type` must be `git`. `runner.type` must be `docker` for real runs.
- `runner.docker.composeFiles` non-empty; `projectPrefix` non-empty.
- `entry.run` and `entry.stop` non-empty. Each entry string runs via
  `sh -c` with `cwd=<worktree>`, `env=<allocated ports>`.
- `project.worktreeBase`: relative resolves against the repo root
  (the directory containing `wrk3.yaml`);
  absolute passes through.

### Port mapping (`ports.base` → `.env` vars)

Only include names your stack actually binds; extra names are harmless.
`app` is required (derived URLs and the `status` APP column build from
it). Every name becomes `<NAME>_PORT` (uppercased, non-alphanumerics →
`_`): `app` → `APP_PORT` (+ `BASE_URL`, `WEBHOOKS_BASE_URL`,
`ALLOWED_WS_ORIGINS` = `http://localhost:<app>`), `web` → `WEB_PORT`.

Three rules for parallel safety (all verified in §0):

1. The compose files **must** consume these vars for host-port bindings
   (e.g. `"${APP_PORT:-8000}:8000"`). If a compose file hardcodes a
   host port, either parameterize it first (§1) or accept the conflict and say so.
2. The compose files **must not** set `container_name:` — it is global on
   the daemon and collides across worktrees (`up` fails fast naming the
   file/services). Delete it (§1); compose generates `<project>-<service>-1`.
3. Two configs sharing one host need distinct `ports.base` offsets or steps,
   otherwise worktree 0 of project A collides with worktree 0 of project B.

## 4. Validate (read-only first)

```bash
wrk3 fetch    # git fetch --prune + list origin/* refs (run inside the repo)
wrk3 status   # renders empty table on a fresh config
# or from anywhere: wrk3 -f <path-to-yaml> fetch
```

Both must exit 0. Typical failures and fixes:

| Error | Fix |
| ----- | --- |
| `project.worktreeBase must not be empty` | Set `project.worktreeBase` (e.g. `.worktrees`). |
| `unknown source/runner type` | Must be `git` / `docker`; check spelling. |
| `runner.docker.composeFiles must list at least one` | Add the compose file found in step 1. |
| `entry.run must not be empty` | Fill `run` and `stop` from repo docs. |
| `load config ... no such file` | Wrong `-f` path or no `wrk3.yaml`/`wrk3.yml` above cwd. |
| `sets container_name for service(s)` | §0 miss — delete `container_name:` from the named file/services. |

## 5. Live verification (needs user approval)

`add` creates real worktrees; `up` boots real containers. Confirm before
running, especially on heavy stacks (double `setup` can take minutes —
flag this when §0 noted large images or slow seeds).
Bare `add` opens the interactive branch picker; bare `up`/`down` apply
to all worktrees.

```bash
wrk3 add <branch>            # worktree + ports + .env (or bare `add` to pick)
wrk3 up                      # setup → compose up → run (all worktrees)
wrk3 status                  # expect running + distinct ports
wrk3 down
wrk3 remove --all            # compose down -v + worktree remove
```

For a second parallel instance, `add`/`up` another branch and confirm
`status` shows distinct ports and distinct `<prefix>-<slug>` compose projects.

## 6. Hand off

- Show the user the final config path, the §0 verdict, and `status` output.
- If the config is meant to be shared, copy it to `wrk3.yaml.example`
  shape — never commit personal local configs.
- Never commit `wrk3.<name>.local.yaml`, `.wrk3-state.json`,
  `.worktrees/`, or `.env` files.
