# Prompt: adapt this ci.yml to my repo

Paste this into ChatGPT, Claude.ai, Gemini or a local model, then paste your files where marked.

---

You are helping me adapt a GitHub Actions workflow for a pnpm workspaces monorepo. I will paste a template `ci.yml` and the files that describe my repo. Adapt the template to my repo. Do not invent package names, scripts or versions: if something is missing, ask me.

Rules to keep:

1. `pnpm/action-setup` must come before `actions/setup-node` when `setup-node` uses `cache: "pnpm"`. The cache step runs the pnpm CLI to compute its key, so pnpm must already be on PATH.
2. Set the pnpm version to the `packageManager` field in my root `package.json`. If it is missing, tell me and suggest adding it.
3. Use a Postgres service container with a `pg_isready` health check. Use the `pgvector/pgvector` image only if my app uses the pgvector extension; otherwise use the plain `postgres` image with the same major version as production.
4. Run in CI only the Playwright specs that need nothing beyond the web app and the database. Specs that need a background worker, object storage or a live third-party API stay out of CI. Tell me which of my specs fall on each side, and remind me to run the manual set somewhere else.
5. Every `env:` value must be a CI-only placeholder. Real secrets go in `${{ secrets.NAME }}`.
6. Make `--filter` names match my workspace packages. Make the lint, typecheck, test, build and migrate commands match my `package.json` scripts.
7. Make `node-version-file` point at a file that exists in my repo (for example `.node-version`).
8. If I use Turborepo or Nx, say what to add; do not add it unless I ask.

Output: the full adapted `ci.yml`, then a short list of every change and why, then anything you could not verify.

My template `ci.yml`:

[paste templates/ci.yml]

My root `package.json`:

[paste]

My `pnpm-workspace.yaml`:

[paste]

My package scripts for the web app and database package:

[paste]

My Playwright spec files and what each one needs (worker, storage, live API, or nothing):

[paste]
