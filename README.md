# renovate-config

Centralized [Renovate](https://docs.renovatebot.com/) configuration presets for
`@schubydoo`'s repositories, consumed by a [self-hosted Renovate CE](https://docs.renovatebot.com/getting-started/running/)
instance running as the `renokeeper[bot]` GitHub App.

The bot is **cron-driven — webhooks are not installed yet**, so config changes here and
manual rebase requests are both picked up on the next scheduled run rather than immediately.
Self-hosting is what makes `postUpgradeTasks` usable (the commands must be allow-listed via
`RENOVATE_ALLOWED_POST_UPGRADE_COMMANDS`); the Mend-hosted app cannot run them.

Every tracked repo carries a one-line `renovate.json` that extends a named preset
here, so all dependency policy is edited in **one place**:

```json
{ "extends": ["local>schubydoo/renovate-config:<repo>"] }
```

## Layout

| File | Purpose |
| --- | --- |
| `default.json` | Shared base policy every named preset extends (`local>schubydoo/renovate-config`). |
| `<repo>.json` | Per-repo preset: `extends` the base and adds that repo's custom managers, holds, and overrides. |

## What the base policy does

- Extends `config:best-practices` (pins Docker + Action digests, config migration,
  npm min-release-age, weekly lock maintenance) plus `security:openssf-scorecard`
  and `mergeConfidence:all-badges`.
- Timezone `America/Los_Angeles`; branches refreshed daily (`before 6am`).
- **Auto-merges** patches, digest re-pins, and stable (`>=1.0`) minor updates.
- **Holds for review** all majors and `0.x` minors (they sit in the PR queue).
- GitHub Actions collapse into a single auto-merged `github-actions` PR.
- Custom/regex-manager (vendored / hand-pinned) bumps never auto-merge by default —
  a manual re-vendor heads-up — unless a repo opts a trusted datasource back in.
- The dependency dashboard is disabled fleet-wide.

## Changelog rate limiting

`default.json` references a `github.com` read-only PAT via
`{{ secrets.RENOVATE_GITHUB_COM_TOKEN }}` to lift the changelog-fetch rate limit. The value is
injected into the self-hosted bot's environment (`RENOVATE_SECRETS`) and is never stored in
this repo.

## Validation

CI runs `renovate-config-validator --strict` on every preset. Run it locally with:

```
npx --yes --package renovate -- renovate-config-validator --strict
```
