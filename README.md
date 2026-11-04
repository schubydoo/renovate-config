# renovate-config

Centralized [Renovate](https://docs.renovatebot.com/) configuration presets for
`@schubydoo`'s repositories, consumed by a [self-hosted Renovate CE](https://docs.renovatebot.com/getting-started/running/)
instance running as the `renokeeper[bot]` GitHub App.

The bot runs on **cron plus inbound GitHub webhooks**. The App webhook reaches the self-hosted
instance through a Cloudflare Tunnel. A webhook triggers Renovate only for events in the repo that
emitted them, so a preset edit merged here does not re-evaluate the downstream repos that extend it.
Those repos pick it up on their own events or the next scheduled run, not immediately.
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
- **Auto-merges** patches, digest re-pins, and minor updates, `0.x` minors included.
- **Holds for review** all majors (they sit in the PR queue).
- GitHub Actions collapse into a single auto-merged `github-actions` PR.
- Custom/regex-manager bumps follow the same rules. Real vendored items (vendored assets,
  hand-pinned binaries) carry an explicit per-repo hold and the `vendored` label.
- The dependency dashboard is disabled fleet-wide.

## Changelog rate limiting

`default.json` references a `github.com` read-only PAT via
`{{ secrets.RENOVATE_GITHUB_COM_TOKEN }}` to lift the changelog-fetch rate limit. The value is
injected into the self-hosted bot's environment (`RENOVATE_SECRETS`) and is never stored in
this repo.

## Validation

CI runs `renovate-config-validator --strict --no-global` on every preset, inside the Renovate
container image pinned in `.github/workflows/validate.yml`. Renovate updates that pin.
`--no-global` checks the files as repo configuration, so a global-only option in a preset fails.
To run the same check locally with Docker, use the same image:

```
img=$(sed -n 's/^ *image: *//p' .github/workflows/validate.yml)
docker run --rm -v "$PWD:/work:ro" -w /work "$img" renovate-config-validator --strict --no-global $(git ls-files '*.json')
```

The file list is required. Without it the validator checks only `renovate.json`, reports
success and exits 0, silently skipping every preset.

## Webhook IP list sync

A Cloudflare rule blocks every webhook source that is not in the `github_hooks` IP list.
The `sync-github-hook-ips` workflow runs daily and copies the `hooks` ranges from
`https://api.github.com/meta` into that list. It writes nothing when the two match. It refuses
to write when GitHub returns no ranges, or fewer than half of the ranges that the list holds.

The workflow needs the `CF_LISTS_TOKEN` secret (a Cloudflare API token with
"Account Filter Lists: Edit" only) and the `CF_ACCOUNT_ID` and `CF_LIST_ID` repo variables.
