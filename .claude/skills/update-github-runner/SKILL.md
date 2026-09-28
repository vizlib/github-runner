---
name: update-github-runner
description: Sync this fork (vizlib/github-runner) with upstream actions/runner and cut a matching custom release. Use whenever asked to "update the runner", "sync with upstream", "check for a new runner version", or "release the latest runner version". Mirrors https://github.com/vizlib/terraform-aws-fiplana/blob/main/docs/runbooks/github-runner-release.md and docs/upstream.txt in this repo.
---

# Update GitHub Runner (fork sync + release)

This repo is Vizlib's fork of `actions/runner`. Custom releases are cut by
rebasing `main` onto upstream `main` and then tagging a release that matches
the upstream version. This skill automates that end to end.

Always determine which **mode** applies before doing anything (see below),
since the destructive step (force-push to `main`) is only appropriate when a
human is present to confirm it.

## 0. Setup check

```bash
git remote -v
```

If `upstream` isn't configured:

```bash
git remote add upstream https://github.com/actions/runner.git
```

## 1. Check whether an update is needed

```bash
git fetch upstream --tags
git fetch origin

LATEST_UPSTREAM=$(git tag -l 'v*' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -1)
CURRENT=$(git show origin/main:src/runnerversion)
```

Compare `LATEST_UPSTREAM` (strip the leading `v`) against `CURRENT`
(`origin/main`'s `src/runnerversion`) and against `origin/main`'s
`releaseVersion`:

```bash
git show origin/main:releaseVersion
```

- If `src/runnerversion` and `releaseVersion` both already equal the latest
  upstream version → **up to date, nothing to do.** Report this and stop.
- If `src/runnerversion` == latest upstream but `releaseVersion` differs →
  a sync happened but the release was never cut. Skip to step 3.
- Otherwise → a new upstream version exists. Continue to step 2.

## 2. Sync fork with upstream

```bash
git checkout main
git pull origin main --ff-only   # or --rebase if origin has moved
git rebase upstream/main
```

**Conflict handling — do not guess blindly.** This fork intentionally strips
upstream's community/CI-management workflows and files that don't apply to a
private fork (issue templates, dependabot config, stale-bot, close-bugs-bot,
close-features-bot, docker-buildx-upgrade, dotnet-upgrade, codeql.yml — see
commit `fd9d20e8` "chore: allow to build own fork"). When upstream modifies
one of these files after we deleted it, the rebase will hit a modify/delete
conflict. In that case:

1. Diff upstream's version of the file against what it looked like when we
   deleted it (`git show <upstream-commit>:<path>` vs `git show
   fd9d20e8^:<path>`). If it's a routine version bump / no functional change
   relevant to us, resolve by keeping it deleted (`git rm -f <path>`).
2. If upstream added genuinely new, relevant behavior to a file we deleted,
   stop and ask a human before resolving — don't silently drop new upstream
   functionality.
3. For any other conflict (not one of the known intentionally-removed
   files), stop and ask a human. Do not resolve unfamiliar conflicts alone.

After resolving each conflict: `git add/rm <file>` then `git rebase --continue`.

Verify the rebase fully caught up:

```bash
git merge-base --is-ancestor upstream/main main && echo "synced" || echo "NOT fully synced"
cat src/runnerversion
```

### Mode A — Interactive (human present)

```bash
git push origin main --force
```

This is a force-push to a shared branch — always confirm with the user
before running it, even if they invoked this skill directly.

### Mode B — Unattended / automated (e.g. from a scheduled job)

Never force-push directly. Instead:

```bash
git checkout -b sync/upstream-$(cat src/runnerversion)
git push origin HEAD
gh pr create --repo vizlib/github-runner \
  --base main \
  --title "Sync with upstream actions/runner $(cat src/runnerversion)" \
  --body "Automated upstream sync. Rebases main onto upstream/main $(git rev-parse upstream/main). Review the diff, then merge with **Rebase and merge** (not squash) to preserve upstream commit hashes, and re-run this skill's release step afterwards."
```

Stop here in Mode B — do not cut the release until the sync PR is merged,
since the release step must run against the real `main` after merge.

## 3. Cut the release

Only after `main` (local, matching `origin/main`) has the target version in
`src/runnerversion`:

```bash
git checkout main
git pull origin main --rebase
git branch -D release 2>/dev/null || true

export TAG=$(cat src/runnerversion)
git tag -d "v${TAG}" 2>/dev/null || true
git checkout -b release

echo "${TAG}" > src/runnerversion
echo "${TAG}" > releaseVersion
git commit -am "chore: release of ${TAG}"
git tag -a "v${TAG}" -m "chore: release of ${TAG}"
```

### Mode A — Interactive

```bash
git push origin "v${TAG}"
```

### Mode B — Unattended / automated

Pushing a release tag directly is also unattended-unsafe (it immediately
triggers the public release build/publish). Instead open a PR carrying the
version-bump commit and let a human push the tag after merge:

```bash
git push origin release:release-${TAG}
gh pr create --repo vizlib/github-runner \
  --base main \
  --head release-${TAG} \
  --title "chore: release of ${TAG}" \
  --body "Bumps src/runnerversion and releaseVersion to ${TAG} to match upstream. After merging, push tag v${TAG} to trigger the release build (see docs/upstream.txt)."
```

## 4. Verify (interactive mode only, after pushing the tag)

```bash
gh run list --repo vizlib/github-runner --limit 3
gh run view <run-id> --repo vizlib/github-runner
gh release view "v${TAG}" --repo vizlib/github-runner
```

Confirm: `check` job passes, all platform builds succeed (especially
`win-x64` / `win-arm64` — these are what Vizlib's self-hosted Windows
runners consume), `release` job publishes artifacts, and the tag shows up
under [releases](https://github.com/vizlib/github-runner/releases).

## Rollback

If a bad release was tagged and pushed:

```bash
git push origin ":refs/tags/v${TAG}"
git tag -d "v${TAG}"
```

Then delete the GitHub Release from the Releases page if one was created,
and re-run step 3 with the correct version.
