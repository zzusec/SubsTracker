# Production Deployment

Production: https://subtracker.hx10.com/admin/dashboard

- Upstream: `wangwangit/SubsTracker`, branch `master`.
- Deployment fork: `zzusec/SubsTracker`, branch `master`.
- Worker: `substracker`.
- Configuration: `wrangler.production.toml`. Do not use `deploy:safe` or `setup` against production: those commands can select or create a different KV namespace.
- The existing KV namespace, custom domain, compatibility date, and 12-minute reminder schedule are retained. The six-hour code update schedule is separate from subscription reminders.

## Automatic Updates

`.github/workflows/sync-upstream.yml` runs at `23 */6 * * *` UTC: approximately 02:23, 08:23, 14:23, and 20:23 Asia/Shanghai. GitHub can delay scheduled runs.

The workflow merges upstream application changes, preserves fork-owned workflows and production configuration, runs lint, tests, and a dry run, then publishes and deploys through Wrangler. It does not use Cloudflare Workers Builds. The latter returned account restriction error `8000036` during setup; the regular Worker deployment API was verified to work.

The `CLOUDFLARE_API_TOKEN` repository secret is required. Use a token scoped to the `substracker` Worker and the `hx10.com` route, with Individual Workers Editor and Workers Routes Write permissions. No local OAuth session or GitHub personal token is uploaded to Actions.

The deployed `SOURCE_COMMIT` variable identifies the production Git commit. The `production-verified` tag moves only after the full deployment and site checks succeed. A run skips deployment only when both markers match its commit; a failed deployment is retried on the next run even when upstream has no additional changes. Application merge conflicts, failed tests, and API errors fail visibly in Actions rather than replacing production with unvalidated code. CI-only merge conflicts retain the fork's CI files.

A monthly empty keepalive commit prevents GitHub's 60-day public-repository schedule inactivity disablement. The inherited `Deploy` workflow is disabled; `Sync Upstream` owns production updates.

## Manual Operation

Check recent runs:

```sh
gh run list --repo zzusec/SubsTracker --workflow sync-upstream.yml
```

Check upstream now:

```sh
gh workflow run sync-upstream.yml --repo zzusec/SubsTracker --ref master
```

Force a fresh validation and deployment, including when upstream is unchanged:

```sh
gh workflow run sync-upstream.yml --repo zzusec/SubsTracker --ref master -f redeploy=true
```

For an authorized local deployment, use the lockfile-pinned dependencies and the production configuration:

```sh
npm ci
npm run lint
npm test
npx wrangler deploy --config wrangler.production.toml --dry-run
npx wrangler deploy --config wrangler.production.toml --keep-vars --var "SOURCE_COMMIT:$(git rev-parse HEAD)"
```

## Data and Rollback

Before the initial upgrade on 2026-10-27, the old Worker, its settings and schedules, and every KV value were saved under the owner's private local state directory `~/.local/state/subtracker/backups/20261027-230124/`. This directory contains credentials and subscription data; never commit or upload it to a public artifact.

The v3 migration replaces the old `subscriptions` array with `sub:{id}` records and `sub_index`. Its temporary `subscriptions_v2_backup` expires after seven days. Four production subscriptions, every original field, the config bytes, and four reminder rules were independently verified after migration.

Rolling back Worker code does not roll back KV data. Returning to the pre-v3 Worker also requires restoring the old subscription representation from the private backup. Review any post-upgrade edits before restoring data; do not blindly overwrite current records.

Future code updates do not automatically create offline KV snapshots. Back up data before any future schema migration. A successful deployment or HTTP 200 alone does not prove data integrity: verify authenticated version, subscription records, dashboard statistics, and required static assets.
