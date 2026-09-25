# CLAUDE.md

Guidance for Claude Code (claude.ai/code) working in this repository.

## What this is

A composite GitHub Action that deploys to GCP. Everything lives in `action.yml` —
there is no build step and no JavaScript, despite the `package.json`. That
`package.json` exists only to carry the semantic-release toolchain.

The `cloudrun` path does three things:

1. `envsubst` the caller's service definition (default `./.gcp/service.yaml`),
   substituting `SERVICE_NAME`, `PROJECT_ID`, `IMAGE`, `CLOUD_RUN_SA`,
   `ENVIRONMENT` and `VPC_ACCESS_CONNECTOR`.
2. Stamp `brandlive.com/deploy-id` into `spec.template.metadata.annotations`.
3. Hand the rendered file to `google-github-actions/deploy-cloudrun@v2` as
   `metadata:`, which runs `gcloud run services replace`.

### Why step 2 exists (BDEV-5770)

`gcloud run services replace` is declarative. When the rendered spec is
byte-identical to what is already deployed, **Cloud Run creates no new revision
and the command still exits 0** — in about three seconds, with the workflow
green. A redeploy of an unchanged image therefore did nothing at all, which
meant there was no way to restart a Cloud Run service from CI. That cost 18
hours during BDEV-5769.

Stamping the run id into the revision template guarantees the spec differs, so
every deploy rolls. Verified against a real dev service: same annotation value
produces no revision, a changed value produces one, and Cloud Run accepts and
persists the custom annotation.

The stamp uses an `awk` insertion that assumes the caller's YAML has
`spec.template.metadata.annotations` with `annotations:` at six-space indent.
It fails the job loudly if it cannot find that block, rather than silently not
stamping. If you change that logic, test it against every caller's real
`service.yaml`, not just one.

## Releases: conventional commits are mandatory

`.github/workflows/release.yaml` runs `npx semantic-release` on every push to
`main`. Versions come entirely from commit messages, using the default angular
preset:

| Commit subject | Result |
| --- | --- |
| `fix: …` / `perf: …` | patch |
| `feat: …` | minor |
| any type with a `BREAKING CHANGE:` footer | major |
| `docs: …`, `chore: …`, `refactor: …`, `ci: …`, `test: …`, `style: …` | **no release** |
| anything not matching the conventional format | **no release** |

**A subject that starts with a Jira key is not a conventional commit.** Much of
the Brandlive org requires PR titles like `BDEV-1234 do the thing`. This repo
has no Jira title gate — the only workflow here is `release.yaml` — and that
format silently produces no release:

```
[@semantic-release/commit-analyzer] › ℹ  The commit should not trigger a release
[semantic-release] › ℹ  Analysis of 2 commits complete: no release
```

The workflow still succeeds, so nothing fails and no tag appears. This happened
on PR #7. Put the ticket reference in the commit body or the scope instead:

```
feat(action.yml): stamp a deploy id so every deploy rolls a new revision

Refs BDEV-5770
```

PRs here are merged with a **merge commit**, not squashed, so semantic-release
reads the commit subjects from the branch. The PR title is not what matters —
the commits are. `git-cz` is available if you want the prompt.

### If a release was missed

The commit is already on `main` and cannot be reworded without rewriting
history. Cut it with an empty commit carrying the right subject:

```bash
git commit --allow-empty -m "feat(action.yml): <what the merged change did>"
git push origin main
```

Pick the type that matches the change that actually landed, not the empty
commit — `feat` for a minor bump, `fix` for a patch.

## Consumers pin exact tags

There is no moving `v2` tag. Every caller pins an exact version, so a release
does nothing until each repo's pin is bumped:

| Repo | Call sites |
| --- | --- |
| greenroom-microservices | 5 (beehive api + timer, content-service, ingress-api, spielberg) |
| api-cron | 1 |
| dashboard-API | 1 |
| live-monitor | 1 |
| datadog-agent | 1 |
| meet-recordings | 1 |

Cutting a release is therefore only half the job. Budget a pin-bump PR per
consumer repo, and remember the greenroom repos deploy real production
services.

## Testing a change

There is no test suite — `npm test` deliberately exits 1. Validate deploy
changes against a real dev Cloud Run service before tagging:

```bash
gcloud run services describe dev-beehive-timer-us-west1-cr \
  --region us-west1 --project greenroom-372217 --format=export > /tmp/svc.yaml
# apply your transform to /tmp/svc.yaml, then
gcloud run services replace /tmp/svc.yaml --region us-west1 --project greenroom-372217
gcloud run revisions list --service dev-beehive-timer-us-west1-cr \
  --region us-west1 --project greenroom-372217 --limit 3
```

`--format=export` round-trips cleanly through `services replace`. Restore the
service by replacing with the unmodified export when you are done.
