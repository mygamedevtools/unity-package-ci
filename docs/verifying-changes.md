# Verifying a change

Merging a `fix:` or `feat:` here moves `v1`, and every package runs the change on its next CI run. actionlint only checks syntax, and this repository's own CI never runs `test.yml`, `packaging.yml` or the versioning workflows. So before merging any change to `.github/workflows/`, Dependabot bumps included (majors most of all), run it from a package repository against the exact commit, without publishing anything.

It takes about ten minutes of work plus the runs. The examples use scene-loader. fixed-input works the same way, since its callers have the same shape.

## 1. Snapshot the package repository

```sh
REPO=mygamedevtools/scene-loader
snapshot() {
  gh release list -R "$REPO" --limit 1000 --json tagName,isDraft,isPrerelease,publishedAt
  git ls-remote --heads --tags "https://github.com/$REPO.git" | grep -v 'refs/heads/tmp/'
}
snapshot > before.txt
```

This records every release, tag (including `upm/<version>`) and branch head (including `upm` and `main`).

## 2. Make a temporary branch that calls the commit under test

Get the commit: `gh pr view <number> -R mygamedevtools/unity-package-ci --json headRefOid -q .headRefOid`. Pin that SHA rather than the branch name, since Dependabot force-pushes its branches. The workflows call each other with `./`, so pinning the top-level call pins the whole set.

In a clone of the package repository, create `tmp/verify-<topic>` from `main`. Use the `tmp/` prefix: no workflow in the package repositories runs on a push to it, and the generated release config treats `feat/*` and `fix/*` as prerelease branches.

Edit only these two files, and never `release.yml`, which releases on every push to `main`:

- `test.yml`: change `@v1` to `@<sha>`. Tests publish nothing; a dispatched run uploads coverage to Codecov under the temporary branch.
- `release-preview.yml`: replace the job with these three. As committed, it calls `versioning-manual.yml` without `dryRun`, which defaults to `false`, so dispatching it unchanged creates a real prerelease and publishes it.

```yaml
# TEMPORARY (tmp/verify-<topic>): every job is non-publishing.
jobs:
  create-preview-release:
    name: Preview Release
    uses: mygamedevtools/unity-package-ci/.github/workflows/versioning-manual.yml@<sha>
    with:
      bumpType: ${{ inputs.bumpType }}
      dryRun: true
      package-path: Packages/com.mygamedevtools.scene-loader
      unitypackage-export-method: PackageExporter.ExportPackage
    secrets: inherit

  verify-semver-dry-run:
    name: Semver Dry Run
    uses: mygamedevtools/unity-package-ci/.github/workflows/versioning-semver.yml@<sha>
    with:
      dryRun: true
      blockReleaseCommit: true
      package-path: Packages/com.mygamedevtools.scene-loader
      unitypackage-export-method: PackageExporter.ExportPackage
    secrets: inherit

  # The two versioning jobs skip packaging when nothing is released, so call it directly.
  verify-packaging:
    name: Packaging (verify only)
    uses: mygamedevtools/unity-package-ci/.github/workflows/packaging.yml@<sha>
    with:
      version: 0.0.0-verify.1
      publish: false
      package-path: Packages/com.mygamedevtools.scene-loader
      unitypackage-export-method: PackageExporter.ExportPackage
    secrets: inherit
```

Drop the jobs that the change can't affect. Read the diff once more for `@<sha>`, `dryRun: true` and `publish: false`, then commit and push the branch. Don't open a PR.

## 3. Run it

```sh
BRANCH=tmp/verify-<topic>
gh workflow run test.yml -R "$REPO" --ref "$BRANCH"
gh workflow run release-preview.yml -R "$REPO" --ref "$BRANCH" -f bumpType=patch
gh run list -R "$REPO" --branch "$BRANCH"
gh run watch <run-id> -R "$REPO" --exit-status
```

## 4. Judge each changed action

Call each changed action **verified** (its step ran and passed), **failed**, or **not exercised**. These runs reach:

| Run | Exercises | Can't reach |
|---|---|---|
| 🧪 Tests | `actions/checkout`, `dorny/paths-filter`, `actions/cache`, `game-ci/unity-test-runner`, `codecov/codecov-action`, `detect-unity-versions.yml` | `jayqi/failed-build-issue-action` (scheduled runs only) |
| Packaging (verify only) | `actions/checkout`, `actions/cache`, `game-ci/unity-builder`, `actions/upload-artifact`, the `upm` subtree split, signing and its check | pushing `upm` and its tag, attaching assets to a release |
| Semver Dry Run | `actions/checkout`, the release config, `cycjimmy/semantic-release-action` installing and loading its plugins | the version analysis and the action's outputs: semantic-release stops at once because a `tmp/` branch isn't a release branch |
| Preview Release | the prerelease version calculation | creating the release, and packaging through it |

Neither run touches this repository's own `release.yml`. It runs only after the merge, so watch that first release when a bump changes an action it uses.

Download the packaging artifacts and check them when the change touches packaging, e.g. that the `.unitypackage` opens and has the samples under `Samples~/`.

Put the run links and verdicts in the PR, as in #17.

## 5. Confirm nothing was published, then clean up

```sh
snapshot > after.txt && diff before.txt after.txt   # must print nothing

for run in $(gh run list -R "$REPO" --branch "$BRANCH" --json databaseId -q '.[].databaseId'); do
  gh api "repos/$REPO/actions/runs/$run/artifacts" -q '.artifacts[].id' |
    xargs -I{} gh api -X DELETE "repos/$REPO/actions/artifacts/{}"
done
gh cache delete --all --ref "refs/heads/$BRANCH" -R "$REPO" --succeed-on-no-caches   # never without --ref
git push origin --delete "$BRANCH"
```

Keep the runs: the PR links to them.

## Dependabot

- Every pin is a major tag, so every bump is a major. All actions share one PR, except `game-ci/unity-test-runner`, which gets its own because `detect-unity-versions.yml` hard-codes the image tag it pulls (see [`.github/dependabot.yml`](../.github/dependabot.yml)).
- The commit prefix is `fix`, so merging a bump releases a patch and moves `v1`.
- To hold an action back, fix its pin in its own PR and add an `ignore` rule to `dependabot.yml`. The rule doesn't change a group PR that is already open: after #17 merged, Dependabot rebased #16 and kept the bump it now ignores, and `@dependabot recreate` got no response. Push a commit to the Dependabot branch that restores the pin instead.

Worked example: #16 bumped `game-ci/unity-builder` to v6. In the [dry run from scene-loader](https://github.com/mygamedevtools/scene-loader/actions/runs/44853101163), Export Unity Package failed, because v6 rejects custom build methods. #17 pinned v5 and added the `ignore` rule, with a [passing run](https://github.com/mygamedevtools/scene-loader/actions/runs/44854117962) as evidence. #16 was then fixed by hand and shipped as 1.2.5.
