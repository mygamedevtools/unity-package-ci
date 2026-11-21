# AGENTS.md

Reusable GitHub Actions workflows that every My GameDev Tools Unity package calls. [README.md](README.md) is the public contract: what each workflow does, its inputs and secrets, and how callers use them. Read it first. This file covers how to change the repository safely.

## Every merge ships everywhere

Callers pin `@v1`. A `fix:` or `feat:` merged to `main` cuts a release, and `release.yml` force-moves `v1` to it, so the change reaches every package on its next run. There is no staging step.

- Merges are squash-only, and semantic-release reads the squashed commit's title: the PR title, or the commit's own title when the PR has a single commit. Get both right. `fix:` releases a patch and `feat:` a minor. `ci:` (this repository's own CI) and `docs:` don't release, but `v1` still moves to include them on the next release. So a change to a shared workflow titled `ci:` or `chore:` reaches every package later, with no changelog entry.
- Stay compatible within `v1`. Removing or renaming an input, output or secret, making one required, or changing what a default does breaks callers: that needs a `BREAKING CHANGE:` footer and becomes `v2`. Add new inputs as optional, with a default that keeps today's behavior (like `unitypackage-rename-samples`).
- The README's input and secret tables mirror the `workflow_call` declarations. Update them in the same PR.

## Checking a change

Nothing here runs locally, and there is no test suite.

- Run `actionlint` from the repository root (`brew install actionlint shellcheck`, or see its README). CI runs it, with shellcheck, on every PR.
- For shell logic, copy the step's `run: |` block into a script, export the step's (and job's) `env:` variables with test values, and run it with `GITHUB_OUTPUT=$(mktemp) bash -e script.sh`. A stub `curl` on `PATH` can fake API failures. #11 and #12 did this and pasted the output into the PR as evidence.
- For an end-to-end run, follow [docs/verifying-changes.md](docs/verifying-changes.md): it runs a package repository's callers against your commit without publishing anything. Do it for any change to `.github/workflows/`, including Dependabot bumps.

## Gotchas

- At job level, `uses: ./.github/workflows/…` calls a workflow in this repository at the same commit. That is how the workflows call each other, and it keeps a caller on one consistent set; never call a sibling with `@v1`. At step level, `uses: ./…` resolves against the caller's checkout, so local composite actions don't work here and shared shell is inlined instead.
- Because of that, logic is duplicated. The Prepare Package step (package path resolution from `.releaserc.json`, version bump, `Samples~` path rewrite) appears in all three `packaging.yml` jobs, and the path resolution again in `versioning-semver.yml`. Change every copy.
- Never write `${{ … }}` inside a `run:` script. The runner pastes the value into the source before bash runs, so a quote in an input breaks the script. Pass it through the step's `env:` and use it as a quoted shell variable (with jq, `--arg` / `--argjson`). Don't reuse a name that a step writes to `$GITHUB_ENV`: the Prepare Package steps take the input as `PACKAGE_PATH_INPUT` because they write `PACKAGE_PATH` there.
- `detect-unity-versions.yml` checks for the exact image `game-ci/unity-test-runner@v4` pulls (`ubuntu-<version>-linux-il2cpp-3`). Bumping the runner in `test.yml` can change that tag.
- Version candidates come only from Unity's public release API, which leaves out Extended LTS patches that a Personal license can't open (exit code 198). If the API is unreachable the job fails on purpose; don't add a fallback to the newest image (#11, #12).
- The root `.releaserc.json` releases this repository. The config callers get is generated inside `versioning-semver.yml`.
