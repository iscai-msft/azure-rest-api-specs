# Bidirectional TypeSpec Branch Sync

`main` uses the latest stable TypeSpec packages. `typespec-next` intentionally uses the
`next` TypeSpec packages to find upcoming compatibility issues. This runbook keeps the
branches close while preserving that deliberate package-version difference.

Run this process for each TypeSpec release cycle after the stable package versions are known.

## Non-negotiable rules

- Use real merge commits. Do not use squash merge or rebase merge.
- Keep `main` and `typespec-next` branch history connected. After each phase,
  `git merge-base --is-ancestor <merged-source> <candidate-tip>` must succeed.
- Do not copy files from one branch with `git checkout <ref> -- <path>`. That copies content
  without preserving the source branch's ancestry.
- Do not overwrite the root `package.json` wholesale. The package policy differs by phase.
- Run `npm install` after every root dependency-manifest change and commit the resulting root
  `package-lock.json`.
- Do not start Phase 2 until the Phase 1 PR is approved and landed.
- Mention `@iscai-msft` in every PR opened as part of this process so Isabella is notified.

## Required validation gate

`TypeSpec Validation - All` is the primary gate for this process. It uses three specification
shards in parallel (`shard` 0, 1, and 2) on both Ubuntu and Windows. A PR-triggered run therefore
has six validation jobs.

Before merging either PR:

1. Trigger the workflow against the candidate branch:

   ```bash
   gh workflow run typespec-validation-all.yaml --ref <candidate-branch>
   ```

2. Find the triggered run and inspect every matrix job:

   ```bash
   gh run list --workflow typespec-validation-all.yaml --branch <candidate-branch> --limit 1
   gh run view <run-id>
   ```

3. Require all six jobs to pass. Do not treat one successful shard or operating system as a
   successful validation run.
4. If any job fails, inspect logs for every failed shard and operating system:

   ```bash
   gh run view <run-id> --log-failed
   ```

   Fix every reported specification failure, rerun the complete matrix, and repeat until all jobs
   pass. A failure in one shard does not prove the other shards are clean.

The workflow also runs four times daily against both `main` and `typespec-next`; investigate any
scheduled failure before beginning the sync so the PR does not conceal a pre-existing branch issue.

## Phase 1: merge `typespec-next` into `main`

1. Start from an up-to-date local `main` and create a working branch:

   ```bash
   git fetch origin main typespec-next
   git switch --create <phase-1-branch> origin/main
   git merge --no-ff origin/typespec-next -m "Merge typespec-next into main"
   ```

2. Resolve conflicts with this policy:
   - `specification/`: integrate the intended TypeSpec changes from `typespec-next`.
   - Root `package.json`: keep `main`'s dependency set and update every `@typespec/*` and
     `@azure-tools/typespec-*` package to the latest compatible **stable** release. Never leave a
     TypeSpec package pinned to `"next"` in `main`.
   - Root `package-lock.json`: regenerate with `npm install`.
   - `eng/`, `.github/`, and other tooling: retain `main` unless a concrete
     `typespec-next` change is required by the promoted specification changes.

3. Confirm the root package policy and ancestry:

   ```bash
   git grep -n '"next"' -- package.json
   git merge-base --is-ancestor origin/typespec-next HEAD
   ```

   The first command must not show TypeSpec package dependencies. The second command must succeed.

4. Run `npm install`, commit the lockfile, run the required validation gate, then create a PR
   targeting `main`. Include `@iscai-msft` in the PR body.

5. Wait for approval and landing. Record the landed `main` SHA before continuing.

## Phase 2: merge the approved `main` into `typespec-next`

1. Fetch the `main` commit that includes the Phase 1 PR, then create a branch from the current
   `typespec-next`:

   ```bash
   git fetch origin main typespec-next
   git switch --create <phase-2-branch> origin/typespec-next
   git merge --no-ff origin/main -m "Merge main into typespec-next"
   ```

2. Resolve conflicts with this policy:
   - `specification/`, `.github/`, `eng/`, `.vscode/`, and normal repository content: merge
     `main` normally. Preserve a `typespec-next` change only when it is a deliberate branch-specific
     requirement.
   - Root `package.json`: keep `typespec-next`'s package manifest and its `"next"` TypeSpec
     dependency pins.
   - Root `package-lock.json`: regenerate with `npm install`.

3. Confirm the intended package policy and ancestry:

   ```bash
   git diff --quiet origin/typespec-next HEAD -- package.json
   git merge-base --is-ancestor origin/main HEAD
   ```

   Both commands must succeed. The first proves the root manifest kept the `typespec-next`
   package policy; the second proves `main`'s actual commits are ancestors of the candidate tip.

4. Run the required validation gate, then create a PR targeting `typespec-next`. Include
   `@iscai-msft` in the PR body.

## Landing the history-preserving merge

GitHub squash and rebase merge both flatten the two-parent merge commit and break the ancestry this
process is designed to preserve. If repository settings do not allow merge commits, an authorized
maintainer must fast-forward the reviewed branch tip directly:

```bash
git fetch origin
git push origin <reviewed-tip-sha>:refs/heads/<target-branch>
```

Immediately verify the result:

```bash
git fetch origin <target-branch>
git merge-base --is-ancestor <source-branch-or-sha> origin/<target-branch>
```

The verification command must succeed before closing the tracking issue.
