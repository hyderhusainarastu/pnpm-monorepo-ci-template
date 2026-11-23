# pnpm-monorepo-ci-template

A GitHub Actions workflow template for pnpm workspaces, with the reasoning behind each choice.

## Using this in your project

- Copy `templates/ci.yml` into the target repo as `.github/workflows/ci.yml` and keep the step order: `pnpm/action-setup` must run before `actions/setup-node` with `cache: "pnpm"`.
- Rename `@acme/db` and `web` to the real workspace package names. Check them against `pnpm-workspace.yaml`.
- Copy `examples/.node-version` to the repo root, or change `node-version-file` in the workflow.
- Set the pinned pnpm version (`version:` in `pnpm/action-setup`) to match `packageManager` in the root `package.json`.
- Keep the `pgvector/pgvector` service image only if the app uses pgvector. Otherwise use the plain `postgres` image.
- List only Playwright specs that need nothing beyond the web app and the database. Leave worker and live-API specs out of CI.
- Never put real secrets in the workflow. Use `${{ secrets.NAME }}` for anything CI must reach for real.
- Paste-in version of the adaptation request: `PROMPT.md`.

## Working on this repo

- There is no build and no test suite. The workflow file is checked by GitHub Actions on push.
- Validate YAML changes with any YAML linter before committing.
- Keep `examples/` fictional. Do not add real project names, hosts or credentials.
