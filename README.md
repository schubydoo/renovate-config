# renovate-config

Centralized [Renovate](https://docs.renovatebot.com/) configuration presets for
`@schubydoo`'s repositories, hosted for the [Mend-hosted Renovate app](https://docs.renovatebot.com/mend-hosted/overview/).

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
`{{ secrets.RENOVATE_GITHUB_COM_TOKEN }}` (stored as an org-level secret in the Mend
dashboard, never in this repo) to lift the changelog-fetch rate limit.

## Validation

CI runs `renovate-config-validator --strict` on every preset. Run it locally with:

```
npx --yes --package renovate -- renovate-config-validator --strict
```
