# .github/ — Design notes

CI workflows.

**Read this when:** a workflow that installs dependencies fails on one Node line but not another, when changing which npm writes `package-lock.json`, or when changing what the release cherry-pick decides to pick.

Index of every design note: [DESIGN.md](../DESIGN.md).

---

## The lock file must pass `npm ci` under npm 10 and npm 11 (`workflows/lockfile-npm-compat.yml`)

npm 11 accepts some lock files that npm 10's `npm ci` rejects as out of sync. Renovate and the Sync Lock File workflow (`.node-version`) write the lock with npm 11, but harper supports Node 22, which ships npm 10. harper#3006 auto-merged such a lock: the root `utf-8-validate` 6.x didn't satisfy the `^5.0.2` optional peer of the `ws@7` copies in the react-native subtree, so npm 10 wants a nested 5.x where npm 11 doesn't. harper#3015 removed that root declaration. `lockfile-npm-compat.yml` runs `npm ci --dry-run` on Node 22 and Node 24 for every PR and `main` push, with no path filter so it can be a required check.

When the Node 22 leg fails, remove the version conflict at its source. Regenerating the lock with npm 10 doesn't hold: the next npm 11 write prunes the npm-10-only entries again. npm 10 also marks the `@cbor-extract/*` prebuilt binaries `dev`, so `build-tools/prune-shrinkwrap-dev.mjs` drops them from the published shrinkwrap.

## A release cherry-pick skips a change its branch already has (`workflows/cherry-pick-patch.yml`, `scripts/change-landed.sh`)

Before picking, the cherry-pick job asks `scripts/change-landed.sh` whether one pick of the PR's whole net change onto the release branch would come out empty: `git merge-tree` of that change, with its own base as the merge base, is clean and leaves the release tree unchanged. Only that answer skips the PR, with no branch and no PR. A change that is partly present, reverted, or conflicting picks as before, and so does a check that cannot run. This is the empty-pick test `scripts/apply-picks.sh` already applies to each commit, lifted to the net change. Replaying a landed PR one commit at a time re-applies each intermediate commit against its own final content, which conflicts instead of coming out empty. harper#3067 was the second run of a `demilestoned`/`milestoned` pair re-picking harper#3038 onto the v5.3 the first run had just landed it on.

The net change is `MERGE_BASE..HEAD_SHA` when the job replays the PR's commits, which includes any merge-commit resolutions; a contained change therefore also clears that hold. It is `MERGE_SHA^1..MERGE_SHA` for a PR merged with a merge commit. The fallback that builds from a lone merge commit is not provably the whole change, so it is never checked and keeps its hold. The check runs after the release branch is checked out, so it reads the same `.gitattributes` as the picks. `unitTests/github/cherryPickPatch.test.mjs` runs the shipped job's steps against a fixture origin.
