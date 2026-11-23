# pnpm Monorepo CI Template

- A ready-to-copy `ci.yml` for pnpm workspaces: install, migrate, lint, typecheck, test, build and Playwright E2E.
- Solves three common traps: `pnpm/action-setup` ordering for caching, a pgvector Postgres service container, and a CI-safe versus manual E2E split.
- Works by hand or with any AI assistant: paste `PROMPT.md` into ChatGPT, Claude, Gemini or a coding agent to adapt it to your repo.

**Works with:** GitHub Actions. For adapting the file: ChatGPT, Claude Code, Codex, Cursor, Gemini (see `PROMPT.md` and `AGENTS.md`).

A GitHub Actions workflow for a **pnpm workspaces** monorepo (Next.js-style web app + shared packages, no build-system layer like Turborepo or Nx) that runs install, migrations, lint, typecheck, unit tests, build, and a Playwright E2E pass against a real Postgres + pgvector service container — plus documentation for three things that are easy to get wrong the first time you wire this up.

This is a plain pnpm workspace setup. If your repo uses Turborepo or Nx, you'll want their caching/task-graph steps too — this template deliberately doesn't include any, so it's a clean starting point for a workspace that doesn't use one.

<details>
<summary>Problem: three pnpm-monorepo footguns</summary>

## Problem

Copying a CI workflow from a tutorial or another repo usually gets you 80% of the way to green, then you lose an afternoon to one of a few recurring pnpm-monorepo footguns:

- **A cache-setup step silently fails or falls back to a slow/no-op path** because the tool it needs (`pnpm`) isn't installed yet, but there's no error — just a cache that never hits.
- **A `pgvector`-dependent app works locally but every DB-backed test fails in CI** because the plain `postgres` service image doesn't have the extension, and `CREATE EXTENSION vector` errors out with no obvious next step.
- **A team ends up either skipping E2E in CI entirely, or CI eventually times out / flakes constantly** because someone added a full end-to-end suite that needs a background worker, object storage, and live third-party APIs, and none of that infrastructure exists in the CI runner.

This template is a working `ci.yml` with those three problems already solved, plus the reasoning written down so the next person (including future-you) doesn't have to rediscover it.

</details>

## Quick start

1. Copy `templates/ci.yml` to `.github/workflows/ci.yml` in your repo. (This repo's own `.github/workflows/ci.yml` is only a self-check of the template, not the template itself.)
2. Rename `@acme/db` and `web` to match your actual workspace package names (see `examples/pnpm-workspace.yaml` and `examples/package.json` for the expected shape).
3. Adjust the Playwright spec list on the last step to your own CI-safe specs, or delete that step entirely if you don't have Playwright wired up yet.
4. Replace the placeholder secrets (see [Secrets](#secrets-are-placeholders) below) with whatever your app actually needs.
5. Copy `examples/.node-version` to your repo root (the workflow reads `.node-version`), or change `node-version-file`.
6. **Bump the pinned pnpm version.** `ci.yml` pins `pnpm/action-setup` to `11.14.0`, and `examples/package.json` shows the matching `packageManager` field. Set both to the pnpm version you actually use, and keep them in sync.
7. Want an assistant to do the adapting? Paste [`PROMPT.md`](PROMPT.md) and your files into it.

<details>
<summary>Before / after: the job-ordering trap</summary>

### Before / after: the job-ordering trap

This is the single most common way this kind of workflow breaks. A natural-looking first draft puts `actions/setup-node` first, since that's the "set up the language runtime" step:

```yaml
# Before — looks reasonable, breaks pnpm caching
steps:
  - uses: actions/checkout@v5
  - uses: actions/setup-node@v5
    with:
      node-version-file: ".node-version"
      cache: "pnpm"          # <-- tries to invoke `pnpm` to compute a cache key
  - uses: pnpm/action-setup@v4
    with:
      version: 11.14.0
  - run: pnpm install --frozen-lockfile
```

`setup-node`'s `cache: "pnpm"` option needs to run `pnpm` itself to compute the cache key (it hashes `pnpm-lock.yaml` via the pnpm CLI, not by reading the file directly). At the point that step runs, above, `pnpm` isn't on `PATH` yet — `pnpm/action-setup` hasn't run. This doesn't fail loudly. Depending on the runner and action versions, you get either a hard error deep in `setup-node`'s internals or a cache that's silently never populated, and either way the fix isn't obvious from the symptom.

The fix is just reordering two steps:

```yaml
# After — pnpm is on PATH before setup-node needs it
steps:
  - uses: actions/checkout@v5
  - uses: pnpm/action-setup@v4     # must come first
    with:
      version: 11.14.0
  - uses: actions/setup-node@v5
    with:
      node-version-file: ".node-version"
      cache: "pnpm"
  - run: pnpm install --frozen-lockfile
```

`templates/ci.yml` in this repo has this correct order with a comment explaining why, so it doesn't get "cleaned up" back into the broken order by someone who doesn't know the reason.

</details>

<details>
<summary>What is in <code>ci.yml</code>, section by section</summary>

## What's in `ci.yml`, section by section

### 1. The job-ordering trap (`pnpm/action-setup` before `setup-node`)

Covered above. The rule: **any step whose caching depends on running the package manager itself needs that package manager already on `PATH`.** This generalizes beyond pnpm+Node — the same failure shape shows up with any "cache: <package manager>" option paired with a language-setup action that also wants to install that package manager for you.

### 2. The pgvector Postgres service container

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg17
    env:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app_ci_only
      POSTGRES_DB: app_ci
    ports:
      - 5432:5432
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
```

If your app uses `pgvector` (embeddings, similarity search, etc.), you cannot use the stock `postgres:17` image as your CI service container — `CREATE EXTENSION vector` will fail because the extension isn't installed. Swap the image for `pgvector/pgvector:pg17` (or whichever major version matches your production Postgres) and everything else about a normal Postgres service container stays the same. This one-line image swap is the whole fix; there's no extra install step needed inside the job.

The `--health-cmd pg_isready` block matters too: without a health check, GitHub Actions will happily start running your `pnpm install` / migration steps before Postgres has finished accepting connections, and your migration step fails with a connection-refused error that looks like a config problem but is actually just a race.

### 3. The CI-safe vs. manual E2E split

The E2E step only runs a subset of the full Playwright suite:

```yaml
- name: E2E (CI-safe specs)
  run: |
    ...
    pnpm --filter web exec playwright test landing.spec.ts onboarding.spec.ts security.spec.ts dashboard.spec.ts settings.spec.ts workspace.spec.ts auth.spec.ts status.spec.ts
```

The reasoning: **a spec belongs in CI only if everything it needs is either already running in this job or seeded directly into the database by the test itself.** In this template that means the Next.js-style web app and the Postgres service container — nothing else. Specs that need a background job worker, object storage, or a live third-party API (search, AI, bibliographic lookups, whatever your app's version of "external service" is) don't go in this list.

That's a deliberate trade-off, not an oversight:

- **What you get:** every push runs a deterministic, fast, zero-external-cost E2E pass. Nothing in CI can flake because a third-party API had a bad five minutes, and nothing in CI can rack up API spend.
- **What you don't get:** CI does not prove your worker-dependent or live-API-dependent flows work. Those need a separate, manually triggered (or scheduled) run against a fuller local or staging stack, with real infrastructure behind it. Document that manual run somewhere your team will actually find it — a `docs/` file, your README, wherever your project log lives — so "CI is green" doesn't quietly get treated as "everything works."

If your app has no background worker and no live third-party dependencies, you may not need this split at all — just run your whole suite in CI. The split earns its complexity only once some of your E2E coverage genuinely can't run in a CI runner.

</details>

## Secrets are placeholders

Every `env:` value in `ci.yml` — `AUTH_SECRET`, the `STORAGE_*` values, the Postgres credentials — is a **CI-only placeholder**, not a real secret. They exist so the app can boot and construct its clients (auth, storage, database) inside the CI job without needing your real infrastructure credentials. None of them point at a real service; the storage URL, for instance, points at `localhost:54321`, which nothing in this job is actually listening on. Swap in your own env var names and, for anything CI genuinely needs to hit for real, use encrypted GitHub Actions secrets (`${{ secrets.YOUR_SECRET }}`) instead of a literal value in the workflow file.

## Limitations

- This is a **workflow file**, not a runnable project. There's no actual `apps/web` or `packages/db` behind it — `examples/` shows the *shape* those need to be in (workspace config + package names), not working code. You'll get real errors if you copy `ci.yml` into an empty repo and expect it to pass as-is.
- It assumes pnpm workspaces specifically. If you're using Turborepo, Nx, or another monorepo tool with its own task graph and remote caching, you'll want to add those steps — this template intentionally leaves them out so it isn't tied to any one build-orchestration tool.
- The Playwright spec list, the migration command, and the `--filter` package names are all illustrative. You will need to edit them to match your actual repo before this workflow does anything useful.
- The "CI-safe vs. manual" E2E split assumes you're disciplined about actually running the manual suite somewhere. Nothing enforces that outside of process — this template doesn't include, say, a separate scheduled workflow for the manual suite (though adding one is a reasonable next step if you adopt this pattern).
- No caching strategy is included beyond `setup-node`'s built-in pnpm store cache. If your build step is slow, you'll likely want to add build-output caching on top of this.

## Related

- [incident-runbook-template](https://github.com/hyderhusainarastu/incident-runbook-template)
- [project-memory-file-scaffold](https://github.com/hyderhusainarastu/project-memory-file-scaffold)
- [pre-publication-review-gate-checklist](https://github.com/hyderhusainarastu/pre-publication-review-gate-checklist)
- [cross-session-agent-handoff-prompt-template](https://github.com/hyderhusainarastu/cross-session-agent-handoff-prompt-template)

## Keywords

github actions, pnpm workspace, monorepo, ci, pgvector, postgres service container, playwright, e2e, caching

---

Part of palimnote-templates - more at https://github.com/hyderhusainarastu
