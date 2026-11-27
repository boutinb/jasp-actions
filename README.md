# jasp-actions

centralized location for custom jasp-actions

## Shared R test setup

Unit tests and coverage both use `setup-r-test-env/action.yml`. Change R/package
installation, system dependencies, JAGS setup, cache settings, and jaspTools
setup there so the workflows stay aligned. The action accepts `r-version`,
`packages` (`lockfile` or `latest`), `needs-jags` (`true` or `false`),
`jasp-version` and `jaspsyntax-source` (see below).
Common environment settings are exported through `GITHUB_ENV` and remain
available to the subsequent test or coverage steps. The caller supplies
`GITHUB_PAT`; the action does not accept or record credentials.

jaspTools imports [jaspSyntax](https://github.com/jasp-stats/jaspSyntax), which
is published on the [jasp-stats R-universe](https://jasp-stats.r-universe.dev).
The action adds the universe to the `repos` of R (pak ignores the
`Additional_repositories` field of jaspTools) and installs jaspSyntax in its own
step, after the other R packages so that neither pak nor a cached library replaces
it, and before `renv::install("jasp-stats/jaspTools")` in lockfile mode (renv
ignores that field too, and `RENV_CONFIG_REPOS_OVERRIDE` replaces its repositories).
On Linux it also installs `libopengl0` and `libegl1`, which jaspSyntax needs: the
universe index does not list them, so pak does not install them.

The JASP version the analyses run with is chosen with two inputs, which the reusable
unit-test workflow passes on as `jasp_version` and `jaspsyntax_source`:

- `jasp-version`: `development` (default), or the tag of a JASP release (e.g.
  `v0.99.0.2`). It is exported as `JASPTOOLS_JASP_VERSION`, so
  `jaspTools::setupJaspTools()` installs the jaspSyntax of that release from GitHub
  and takes the html files and data sets of the jasp-desktop commit it was built from.
- `jaspsyntax-source`: where the jaspSyntax of the development version comes from,
  `r-universe` (default) or `github` (jaspSyntax `master`, built in the job).

Empty values mean the defaults, so a module can pass its `workflow_dispatch` inputs
straight through and still get development and r-universe on other events.

The workflows retain their own job matrices, schedule checks, unit tests,
environment artifacts, coverage calculation, and Codecov upload. Coverage
uses the same Linux/lockfile setup as unit tests, including installing JAGS
before setting up jaspTools when requested.

Both workflows reference `jasp-stats/jasp-actions/setup-r-test-env@master`,
following the repository's existing action-reference convention. To test a
branch before merging, point those two action references at the branch too.
A relative `uses: ./setup-r-test-env` would look in the calling module's checkout,
which does not contain this action.

The older `setup-test-env` action remains available for existing callers; it
does not implement the current lockfile/latest-package setup.

## Unit-test environment records

`.github/workflows/unittests.yml` records and uploads the actual R environment
after `jaspTools::setupJaspTools()` and before running tests. Each matrix job
uploads `test-environment-<os>-R-<version>-<lockfile|latest>-attempt-<number>`,
including passing runs so that the last passing environment can be compared
with the first failing one. Separate attempt names preserve records on reruns.

Each artifact contains:

- `packages.csv`: the packages visible through `.libPaths()`, their installed
  versions, library paths, build information, repository, and GitHub remote
  references/commit SHAs when present. If multiple libraries contain the same
  package, only the first version that R would find is recorded.
- `metadata.csv`: key/value records with schema version `1`, UTC collection
  time, module commit, run ID/attempt, caller workflow reference/SHA, matrix
  configuration, R version/platform, and runner OS/architecture/image version.
- `session-info.txt`: `sessionInfo()` and `extSoftVersion()` for R's session
  and external library details. This is not a full system-package inventory.

Recording and uploading are non-blocking diagnostics; their failure does not
prevent tests or change the existing release-gating behavior. A dependency
installation or setup failure before recording will not produce an artifact.
The records use base R and do not install additional packages.
CSV fields should be read as strings (in R, `read.csv(..., colClasses = "character")`)
to preserve commit SHAs and run IDs exactly.

Artifacts expire after **30 days** by default. Callers can set
`environment_retention_days` under the reusable job's `with:` section, within
the repository's artifact retention limit. A dashboard consuming these records
should collect them before expiry and prune its own history to a bounded date
window; artifact expiry does not remove copies already stored by the dashboard.

## Update R wrappers

`.github/workflows/update-wrappers.yml` regenerates the R wrappers of a module (`R/<analysis>Wrapper.R`) and their help files (`man/*.Rd`) from its QML forms, and commits them when they changed. It installs [jaspSyntax](https://github.com/jasp-stats/jaspSyntax) from the [jasp-stats R-universe](https://jasp-stats.r-universe.dev), or from GitHub when the `jaspsyntax_ref` or `syntaxinterface_release` input is set, with the pre-built SyntaxInterface library from its GitHub release, so nothing of JASP is built. Modules that do not set `hasWrappers: true` in `inst/Description.qml` are skipped.

Add this as `.github/workflows/update-wrappers.yml` to a module:

```yaml
name: Update R Wrappers

on:
  push:
    branches:
      - master
    paths: ['inst/qml/**', 'inst/Description.qml', '.github/workflows/update-wrappers.yml']
  workflow_dispatch:

jobs:
  update-wrappers:
    uses: jasp-stats/jasp-actions/.github/workflows/update-wrappers.yml@master
    secrets:
      PUSH_TOKEN: ${{ secrets.REPOS_KEY }}
    permissions:
      contents: write
```

`PUSH_TOKEN` must be allowed to push to the protected `master`; `REPOS_KEY` is the org token the translations already push with. It is only used for the push: the checkout uses the `GITHUB_TOKEN`, so with a bad token the wrappers are still generated and the run fails at the push, saying the token is expired or has no write access. Without `PUSH_TOKEN` the `GITHUB_TOKEN` is used, which cannot push to a protected branch. The commit carries `[skip ci]`, so it does not trigger the version bump or the unit tests again. The inputs `jaspsyntax_ref` and `syntaxinterface_release` pin the jaspSyntax version and the SyntaxInterface release.

To run it locally on a checkout, with jaspSyntax and roxygen2 installed: `Rscript update-wrappers/updateWrappers.R <module>`.
