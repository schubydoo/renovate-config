# renovate-config

Centralized Renovate presets consumed by a self-hosted Renovate CE bot (`renokeeper[bot]`).
Each tracked repo carries a one-line `renovate.json` extending a preset here. See @README.md.

## Critical commands

Validate every preset — **YOU MUST pass the file list**:

```
npx --yes --package renovate -- renovate-config-validator --strict $(git ls-files '*.json')
```

**IMPORTANT:** bare `renovate-config-validator --strict` validates only `renovate.json`, prints
"Config validated successfully", and exits 0. It silently skips every preset. A green run
without file arguments proves nothing.

## Architecture

- `default.json` — base policy that every named preset extends.
- `<repo>.json` — per-repo preset: extends the base, adds that repo's custom managers and overrides.
- The `<repo>.json` files present here are the **complete** fleet. A repo without one is excluded
  deliberately (forks, clones, packaging taps, static sites) — do not propose onboarding it.

## Hard rules

- **`packageRules` order decides behavior.** Later matching rules override earlier ones per field.
  Read the whole array before concluding which rule wins.
- **The `custom.regex` rule is LAST in `default.json`** and sets `automerge: false`. It overrides
  the earlier patch/digest automerge rules for every regex-managed dependency.
- A PR carrying the **`vendored`** label was held by that rule — not by CI, not by a missing
  approval. Check the label before investigating checks.
- **Never edit `default.json` to unblock a single repo.** Add a later rule in that repo's
  `<repo>.json` and scope it with `matchUpdateTypes` so the base hold still applies elsewhere.
- A per-repo `groupName` / `postUpgradeTasks` rule does **not** re-enable `automerge`. Set it explicitly.
- **`postUpgradeTasks` require self-hosting** (commands must be on
  `RENOVATE_ALLOWED_POST_UPGRADE_COMMANDS`). Never suggest the Mend-hosted app.
- **The bot runs on cron plus inbound GitHub webhooks** (App webhook → self-hosted Cloudflare
  Tunnel). A webhook triggers Renovate on events in the emitting repo only — a
  preset edit merged here does **not** re-evaluate the downstream repos that extend it; those re-run
  on their own events or the next scheduled run. So a preset change still lands on the cron cycle —
  expected, not a failure.
- **This repo is public. YOU MUST NOT name private repositories** in any tracked file.

## Workflow

- Run the validator above before every commit; CI runs exactly that command.
- Keep preset edits surgical — add a rule, never restructure the array. Order is semantic, so
  reordering silently changes policy.
- When a Renovate PR misbehaves, read the **Automerge:** line in the PR body first. It states the
  effective decision and saves reconstructing the rule chain by hand.
