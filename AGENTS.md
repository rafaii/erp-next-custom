# AGENT OPERATING RULES

You are helping me design and develop a Real Estate Arbitrage Saas platform. I want all codes to be built and pushed to GitHub. I should be able to just clone the repo to a new installation of ERPNext and all the features should be available on the new instance.

## Repo & Environment:

### Main Repo - DO NOT COMMIT or PUSH THIS REPO

- **Main canonical repo (GIT)**: `~/Development/erpnext`

### Custom App Repo - ONLY USE THIS REPO

- **Custom app canonical repo (GIT)**: `~/Development/erpnext/apps/real_estate_os`
  - It is the custom Frappe app.
  - `frappe`/`erpnext` apps and the bench root are upstream/untracked — never commit them.
- **Bench root (a different git)**: `~/Development/erpnext`
  - Frappe Bench install: `apps/frappe`, `apps/erpnext`, `sites/`, `env/`, `config/`.
  - Bench root has NO `.git`; do NOT `git init` it.
- **GitHub remote**: `https://github.com/rafaii/real-estate.git`
- **Development Environment**: Ubuntu with ERPNext v15 installed at `~/Development/erpnext`

You may change code + project files in the paths above.
Credentials: stored in `.env` (not in git - never commit or expose this).

### Fresh-install / clone procedure (acceptance test)

A new ERPNext install must reproduce everything by:

```bash
bench get-app https://github.com/rafaii/real-estate.git
bench --site <site-name> install-app real_estate_os
```

All DocTypes, fixtures, print formats, roles, scheduler jobs, and portal views
must be self-contained in the app (via `hooks.py` + fixtures). No manual
bench-root or site edits are part of the deliverable.

# SYSTEM DESIGN

Refer to `vault/PLATFORM_STRATEGY.md` for the high-level multi-OS SaaS
platform plan, and `vault/os/REALESTATE_MASTERPLAN.md` for the Real Estate
OS's own PRD, to know what we are trying to build.

# REQUIRED GIT WORKFLOW

Assume multiple coding agents may run in parallel.
All coding agents MUST use isolated branches and worktrees.

## 1 Never work directly on `main`

`main` is the integration branch only. Do not implement tasks directly on `main` or use stash/pop on `main` as the normal working model.

## 2 Agent identity & Branch naming

Use a consistent agent label (`developer-1`, `ui-designer-1`, etc.).
For any task, create a short-lived branch:

- `agent/<agent-name>/<task-slug>` (e.g., `agent/developer-1/override-odoo-login`)

## 3 Worktrees are required

Never share a working directory with another task branch. For each task, create a dedicated git worktree inside the relevant repository:

```bash
cd ~/Development/erpnext/apps/real_estate_os
git fetch origin
git worktree add ./wt-<agent-name>-<task-slug> -b agent/<agent-name>/<task-slug> origin/main
```

## 4 Merge policy & Manual-trigger conditions

Default behavior is AUTOMATED isolated merging. However, manual intervention (STOP and ask Imran) is required if:

1. You touch high-risk/shared-critical files (e.g., configs, database migration files, `vault/INDEX.md`, `vault/DECISIONS.md`).
2. Another active coding agent is working on the same feature area.
3. Your change includes architectural tradeoffs or unclear blast radius
4. You need to create, approve, supersede, or number an ADR.

# VAULT (SINGLE SOURCE OF TRUTH)

Canonical vault path: `~/Development/erpnext/vault`

The canonical vault is authoritative for architecture, implementation plans, module design, and ADRs. Code lives in the app repo; documentation lives in the canonical vault.

The vault is LOCAL-ONLY (not part of the `real_estate_os` app repo). Only the
custom app code is committed to GitHub, per the repo-structure ADR.

## 1. Critical vault rule

If you are working from a git worktree, **DO NOT rely on or write to the worktree-local `vault/` copy**. ALWAYS read and write vault files through the canonical vault path above to prevent conflicting vault history.

## 2. Vault structure

```
vault/
├── INDEX.md                     ← lists every domain + link to its index
├── CHANGELOG.md                 ← append-only log of every change, all domains
├── DECISIONS.md                 ← index of approved ADRs
├── decisions/
│   ├── drafts/                  ← draft-adr-<slug>.md (unapproved, unnumbered)
│   └── 0001-<slug>.md           ← approved ADRs, sequential numbering
├── _templates/
│   ├── feature-template.md
│   ├── adr-template.md
│   └── draft-adr-template.md
└── <domain>/
    ├── <domain>.md              ← domain index: purpose + list of feature files
    └── features/
        └── <feature>.md         ← one file per feature/task
```

Domains: Example - `custom-module`, `workflow-orchestration`, `api-gateway`, `ui`, `infra-aws`, `compliance-security`.

Every domain file lives at `vault/<domain>/<domain>.md` — never as a loose file directly in `vault/` root. `INDEX.md` only links to domain folders, it never duplicates domain content.

Every feature file carries YAML frontmatter (`status: planned|in-progress|done|blocked`, `owner`, `domain`, `created`, `updated`, `related_adr`) so Dataview queries in `INDEX.md` and each domain index can auto-generate live status tables instead of relying on manual updates.

## 3. Before any task

1. Read `vault/INDEX.md` to identify the relevant domain.
2. Read that domain's index, e.g. `vault/ui/ui.md`.
3. Open or create the feature file, e.g. `vault/ui/features/dashboard-widgets.md`.
   - If new: use `_templates/feature-template.md`, detail the feature, write an implementation plan.
   - If continuing: read the existing implementation plan, do the task, check off completed items, update `status` in frontmatter.

## 4. After each coding session (MANDATORY)

Whenever you change code:

1. Update the feature's `<task>.md` in the canonical vault (check off completed items, update `status` frontmatter, bump `updated` date).
2. Append an entry to `~/Development/erpnext/vault/CHANGELOG.md` using the format:
   `YYYY-MM-DDTHH:MM:SSZ | domain | description | author | branch: agent/<agent-name>/<task-slug>`
   - **`description` MUST be 1-2 sentences, no exceptions.** State what changed and,
     if it's a fix, the one-line root cause — nothing else. Every entry needs a
     description; never leave it blank.
   - Investigation detail, verification steps, root-cause narratives, and cleanup
     notes belong in the feature file's Implementation Plan/Acceptance Criteria
     (§3 above), not in CHANGELOG.md. CHANGELOG.md is a terse index of *when*
     something happened and *where* to look (which feature file/ADR), not the
     record of *how* — that record already lives in the feature file.
   - If you genuinely need to note something not captured elsewhere, add it to
     the relevant feature file, not by expanding the CHANGELOG line.
3. If you created a new feature-file implementation plan, or just completed one
   (checked off its last Implementation Plan item / flipped its `status` to
   `done`), update `~/Development/erpnext/vault/IMPLEMENTATION-PLAN.md`
   accordingly in the same session: add a new row for a created plan, or move a
   completed one down to its "Completed" section. Do not let
   IMPLEMENTATION-PLAN.md drift out of sync with the feature files it tracks.

## 5. ADR workflow (DRAFT FIRST)

For non-trivial architectural decisions (e.g., swapping a core workflow engine, changing the provider-abstraction contract):

1. Create a draft ADR first in `vault/decisions/drafts/draft-adr-<slug>.md` using `_templates/draft-adr-template.md`.
2. Summarize the tradeoff in chat and ask Imran for approval.
3. Only upon approval: assign the next sequential ADR number (e.g. `0001`, `0002`, zero-padded, chronological — never reuse or renumber), move the file to `vault/decisions/000N-<slug>.md` using `_templates/adr-template.md`, delete the draft, and add a row to `DECISIONS.md`.
4. Once the ADR is approved, create the implementation plan as a feature file in the relevant domain's `features/` folder (e.g., an ADR about provider-abstraction failover creates `vault/provider-abstraction/features/failover-strategy.md`), and link it from the ADR's Implementation section.

# DEPLOYMENT (VPS)

VPS path: `/home/imranrafai/realestate`
VPS SSH: ssh -i ~/.ssh/id_claude_vps -p 4114 imranrafai@160.238.36.235

## 1 Deployment workflow

Default deployment branch: `main`. Safe isolated work should self-complete end-to-end: commit branch → push branch → merge to `main` → push `main` → SSH to VPS → `git pull origin main` in `apps/real_estate_os` (NOT the outer `/home/imranrafai/realestate` dir — that has no `.git`) → rebuild the custom image → recreate containers → verify.

**CRITICAL: All code changes MUST be made locally, committed, and pushed to GitHub. The VPS pulls from GitHub. Never edit code directly on the VPS — it will be overwritten on next pull.**

**The custom app image is NOT built by `docker compose --build`.** `compose.yaml`'s services reference a fixed, pre-built tag (`${CUSTOM_IMAGE}:${CUSTOM_TAG}` = `realestate-custom:v15.119.3`, no `build:` section) — `--build` is a no-op for it. The actual image build is a separate step:
```
cd /home/imranrafai/realestate/apps && git -C real_estate_os pull origin main && docker build -t realestate-custom:v15.119.3 .
```
(`apps/Dockerfile` copies `real_estate_os/` in and `pip install -e`s it.) Only after that does `docker compose ... up -d --force-recreate` actually pick up the new code.

**The full compose invocation needs ALL of these `-f` files — omitting any is destructive, not just incomplete:**
```
cd /home/imranrafai/realestate && docker compose -p realestate --env-file .env \
  -f frappe_docker/compose.yaml \
  -f frappe_docker/overrides/compose.mariadb.yaml \
  -f frappe_docker/overrides/compose.redis.yaml \
  -f realestate-override.yml \
  up -d
```
- Project name **must** be `-p realestate` — without it, compose defaults to a project name derived from the directory (`frappe_docker`, since that's the first `-f` file's dir), which creates a second, parallel set of containers instead of reconciling the existing `realestate-*` ones.
- `compose.mariadb.yaml` and `compose.redis.yaml` are NOT optional extras — `db`, `redis-cache`, `redis-queue` are only defined in those override files, not in the base `compose.yaml`. **Confirmed the hard way:** running `up -d --force-recreate --remove-orphans` with only `compose.yaml -f realestate-override.yml` correctly recreated backend/websocket/queue/scheduler/frontend, but silently deleted the running `db-1`/`redis-cache-1`/`redis-queue-1` containers as "orphans" (they weren't part of that reduced config). Data survived only because they're named volumes (`realestate_db-data`, `realestate_redis-queue-data`) — recreating the containers against the same volume names via the full `-f` list restored everything with no data loss, but this was recoverable by luck, not by design. **Never pass `--remove-orphans` unless every override file that defines a still-needed service is included in the same command.**
- After recreating `redis-queue`/`redis-cache`, containers that were already running (`queue-short`, `queue-long`, etc.) can end up in a DNS-resolution crash loop (`EAI_AGAIN redis-queue`) if they started before the redis containers finished attaching to the network — `docker restart <name>` on the stuck ones clears it.

**If the change touches `hooks.py`: rebuilding the image and recreating containers is NOT enough.** Frappe caches resolved hooks in Redis per site, independent of the code on disk — a stale `permission_query_conditions`/`has_permission`/etc. entry (or a missing new one) keeps applying until you also run `bench --site <site> clear-cache`. Confirmed the hard way: a permission-hook fix appeared deployed (image rebuilt, containers recreated, site healthy) but the exploit it was meant to close still worked until the cache was cleared.

## 2 Docker Container Setup

docker-compose.yml (on your VPS)
