# Scala GitHub Actions

## Scala Continuous Integration (CI) workflow

Runs tests with coverage, binary compatibility, formatting and Scaladoc checks on every pull request and push on 
"main" branch. Replaces the handwritten `ci.yml` that each project used to carry.

### Setup

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ master ]
  pull_request:

jobs:
  test:
    uses: evolution-gaming/scala-github-actions/.github/workflows/ci.yml@<sha> # v6.4.0
```

Nothing else is required if the project uses the defaults below.

Two things about that snippet are deliberate, both because SonarQube Cloud's quality gate rejects the
alternatives and drops the security rating to C:

* the workflow is pinned to a **full commit SHA**, with the tag in a trailing comment. The snippet
  writes it as `<sha>` on purpose, so this README cannot go stale every time `v6` moves. Resolve the
  current value with:

  ```sh
  gh api repos/evolution-gaming/scala-github-actions/git/tags/$(
    gh api repos/evolution-gaming/scala-github-actions/git/ref/tags/v6 --jq .object.sha
  ) --jq .object.sha
  ```
* there is **no `secrets: inherit`**. It is not needed — `GITHUB_TOKEN` is available to a called
  workflow automatically, and that is what the Coveralls upload uses. Only the optional Sonar scan
  needs a secret.

### Inputs

| input               | default                  | notes                                                                        |
|---------------------|--------------------------|------------------------------------------------------------------------------|
| `scala_versions`    | `'["2.13.18", "3.3.8"]'` | JSON array; becomes the build matrix                                         |
| `java_version`      | `'17'`                   |                                                                              |
| `java_distribution` | `'temurin'`              |                                                                              |
| `test_task`         | auto                     | `testFull` on sbt 2, `test` on sbt 1, read from `project/build.properties`   |
| `clean_task`        | auto                     | `cleanFull` on sbt 2, `clean` on sbt 1, read from `project/build.properties` |
| `sonar`             | `false`                  | run a SonarQube Cloud scan, see below                                        |
| `sonar_project_key` | `<owner>_<repo>`         |                                                                              |
| `sonar_args`        | `''`                     | extra `-D` arguments for the scanner                                         |

Example for a project without `sbt-version-policy` and on a different Scala set:

```yaml
jobs:
  test:
    uses: evolution-gaming/scala-github-actions/.github/workflows/ci.yml@<sha> # v6.4.0
    with:
      scala_versions: '["2.13.18", "3.3.7"]'
```

### Jobs in CI pipeline

All checks are run concurrently! Ideally, we must strive to keep them all green, but, it is allowed to merge PR, if 
some checks are red, for example if code formatting is not introduced, yet. Such red checks must be treated as nudge 
to improve the quality of code in repo! 

* `test-coverage` - runs with disabled disk cache for SBT setup action (`disk-cache: false`) to make sure that 
  test coverage gets run with fully instrumented compilation. The workflow also fails if the produced Cobertura 
  report has no valid lines, so a silently empty report is an error rather than a green build.
  If project has `sonar` integration configured and 
  enabled, then `sonar scan` will get run after coverage reports are uploaded
* `binary-compatibility` - runs [sbt-version-policy](https://github.com/scalacenter/sbt-version-policy/)'s 
  `versionPolicyCheck` task on repo with full history (`fetch-depth: 0`) to make sure that plugin can find the tag 
  for previous version
* `formatting` - runs [scalafmt](https://scalameta.org/scalafmt/)'s `scalafmtCheckRepo` task (requires at 
  least version 2.6.2)
* `scaladoc` - calls `Compile/doc` task to make sure that Scaladocs compile

### SonarQube Cloud

Most repositories should **not** use the `sonar` input. Analysis is done server-side by SonarQube
Cloud's Automatic Analysis, which needs no token, no CI step and no repository configuration — that is
how `kafka-journal` and the other analysed repositories work. The `sonar` input exists for the
CI-based scanner, which is the mutually exclusive alternative.

If you do enable it, it scans once, on the first Scala version, importing scoverage's per-module
reports. Configuration is passed as scanner arguments, so no per-repo `sonar-project.properties` is
needed.

Three prerequisites, all outside this repo:

1. A `SONAR_TOKEN` organization secret. It does not exist yet, so `sonar: true` will fail until it is
   added.
2. The project must already exist on SonarQube Cloud — the scanner reports to a project, it does not
   create one.
3. **Automatic Analysis must be turned off** for that project. It and CI-based scanning are mutually
   exclusive, and Automatic Analysis wins, so the scan will be rejected while it is on.

The scanner also needs the token passed through, which the caller must do explicitly now that
`secrets: inherit` is gone:

```yaml
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Note that the SonarQube Cloud GitHub App creates a check suite on every commit even in repositories
it never analyses, which leaves a check permanently queued and reporting no result. Repositories not
being analysed should have the app removed rather than left in that state.

## Dependency graph workflow

Submits the sbt dependency graph to GitHub. GitHub cannot parse `build.sbt` natively, so a repo
that skips this gets **zero Dependabot alerts** and looks falsely clean. Every repo adopting these
workflows MUST include it.

### Setup

Create `.github/workflows/dependency-graph.yml`, triggered on push to the default branch only:

```yaml
name: Dependency graph

on:
  push:
    branches: [ master ]

permissions:
  contents: write

jobs:
  submit:
    uses: evolution-gaming/scala-github-actions/.github/workflows/dependency-graph.yml@<sha> # v6.4.0
```

Resolve `<sha>` the same way as for the CI workflow above.

The `permissions` block is required. A called workflow can only downgrade the caller's token, never
raise it, so on a repository whose default workflow permissions are read-only the submission fails
without it.

A scheduled [audit workflow](.github/workflows/dependency-graph-audit.yml) in this repository lists
the organization's sbt repos that are missing the caller file and fails while any exist.

### Inputs

| input               | default                                             | notes                                                      |
|---------------------|-----------------------------------------------------|------------------------------------------------------------|
| `java_version`      | `'17'`                                              |                                                            |
| `java_distribution` | `'temurin'`                                         |                                                            |
| `modules_ignore`    | `''`                                                | unpublished modules, with binary version, e.g. `docs_2.13` |
| `configs_ignore`    | `'test integration-test scala-tool scala-doc-tool'` | configurations excluded from the submitted graph           |

Ignore modules that are never published (documentation, integration tests), so their dependencies
do not generate alerts for artifacts nobody consumes:

```yaml
permissions:
  contents: write

jobs:
  submit:
    uses: evolution-gaming/scala-github-actions/.github/workflows/dependency-graph.yml@<sha> # v6.4.0
    with:
      modules_ignore: 'docs_2.13 docs_3 foo-it-tests_2.13 foo-it-tests_3'
```

## Scala Release workflow (v3, v4, v5)

### Setup

To use Scala Release workflow have to set up project:
* add latest [sbt-dynver](https://github.com/sbt/sbt-dynver) plugin
* add Evolution's artifactory plugin [sbt-artifactory-plugin](https://github.com/evolution-gaming/sbt-artifactory-plugin)
* defined command alias `check` which runs code quality checks, for 
  example: [scalafmt](https://scalameta.org/scalafmt/) and binary compatibility check 
  by [sbt-version-policy](https://github.com/scalacenter/sbt-version-policy/):
  ```sbt
  addCommandAlias("fmt", "all scalafmtRepo") // optional: for development
  addCommandAlias("check", "all versionPolicyCheck Compile/doc scalafmtCheckRepo")
  addCommandAlias("build", "+all compile testFull") // optional: for development
  ```
  as very minimum "no-op" placeholder:
  ```sbt
  addCommandAlias("check", "show version")
  ```
* direct `publishTo` to Evolution's artifactory (for publishing artifacts using `sbt-artifactory-plugin`):
  ```sbt
  publishTo := Some(Resolver.evolutionReleases)
  ```
* create `release.yml` file with content:
  ```yaml
    name: Publish Release
    
    on:
      push:
        tags:
          - 'v*'
    
    jobs:
      release:
        uses: evolution-gaming/scala-github-actions/.github/workflows/release.yml@v5
        secrets: inherit
    ```

### Usage

In project's repo:
* push tag as separate activity 
* crate new version tag, like `git tag v1.2.3 -a -m "release v1.2.3"`
* push the tag, `git push origin tag v1.2.3`

The above sequence will start Release workflow, which will:
* run SBT commands `+clean; +check; +all test package` to verify the build
* run SBT command `+publish` to publish packaged artifacts
* if any of above steps will fail, the workflow will remove git tag - improve code and push fix, later tag again
* workflow will auto-generate release notes and will publish them
* go to `Code` and navigate to `Releases`
* review release notes and amend, if required

The `v4` version additionally allows overriding SBT commands used by the release job.
It is especially useful for projects which have mixed Scala versions in submodules and for which the `+`, `+all`
SBT features might not work properly:
```yaml
jobs:
  release:
    uses: evolution-gaming/scala-github-actions/.github/workflows/release.yml@v4
    secrets: inherit
    with:
      # override sbt commands so they don't use "+" because the project uses mixed Scala versions with sbt-projectmatrix
      verify_sbt_command: 'clean; check; all test package'
      publish_sbt_command: 'publish'
```

## Scala Release workflow (v2)

Replaced by `v3` because GitHub excluded SBT from `ubuntu-latest` ([details](https://github.com/actions/setup-java/issues/712#issuecomment-2557396980)). 

## Scala Release workflow (v1)

### Setup

To use Scala Release workflow have to set up project:
* add latest [sbt-release](https://github.com/sbt/sbt-release) plugin
* add Evolution's artifactory plugin [sbt-artifactory-plugin](https://github.com/evolution-gaming/sbt-artifactory-plugin)
* defined command alias `check` which runs code quality checks, for example: [scalafmt](https://scalameta.org/scalafmt/)
  and [scalafix](https://scalacenter.github.io/scalafix/), and binary compatibility check by
  [sbt-version-policy](https://github.com/scalacenter/sbt-version-policy/):
  ```sbt
  addCommandAlias("fmt", "all scalafmtAll scalafmtSbt; scalafixEnable; scalafixAll") // optional: for development
  addCommandAlias("check", "all versionPolicyCheck Compile/doc scalafmtCheckAll scalafmtSbtCheck; scalafixEnable; scalafixAll --check")
  addCommandAlias("build", "all compile test") // optional: for development
  ```
  as very minimum "no-op" placeholder:
  ```sbt
  addCommandAlias("check", "show version")
  ```
* direct `publishTo` to Evolution's artifactory (for `sbt-release` using `sbt-artifactory-plugin`):
  ```sbt
  publishTo := Some(Resolver.evolutionReleases)
  ```
* create `release.yml` file with content:
  ```yaml
    name: Publish Release
    
    on:
      release:
        types: [published]
        branches: [main]
    
    jobs:
      release:
        uses: evolution-gaming/scala-github-actions/.github/workflows/release.yml@v1
        secrets: inherit
    ```

### Usage

On GitHub:
* go to `Releases` page
* press `Draft a new release` button
* make sure that `version.sbt` file in root of project have correct version string, like `1.2.3`
* in `Choose a tag` create a new tag (`v1.2.3`)
* set `Release title` to the same string (`v1.2.3`)
* press `Generate release notes` and review them
* press `Publish release` button
* navigate to `Actions` - there it is possible to follow build's progress

The above sequence will start Release workflow, which will:
* validate consistency of git tag and version
* run SBT commands `clean; +all check test package` to make sure that code quality is good
* run SBT command `+publish` to publish packaged artifacts
* if any of above steps will fail, the workflow will remove git tag and GitHub will mark release notes as `draft` and
  it will be possible to adjust code on main branch and attempt publishing again by navigating to drafted release and
  pressing `Publish release` button again

When release is published, make sure to amend `version.sbt` file with next version string.

## Recommended Scala project setup

### scalafmt

Minimal [configuration](.scalafmt.conf) requires usage of [plugin](https://scalameta.org/scalafmt/) like:
```sbt
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "<latest version>")
```

### scalafix

Minimal [configuration](.scalafix.conf) requires usage of [plugin](https://scalacenter.github.io/scalafix/) like:
```sbt
addSbtPlugin("ch.epfl.scala" % "sbt-scalafix" % "<latest version>")
```

### binary compatibility
Requires usage of [sbt-version-policy](https://github.com/scalacenter/sbt-version-policy/) plugin like:
```sbt
addSbtPlugin("ch.epfl.scala" % "sbt-version-policy" % "<latest version>")
```

### scalac options

Evolution's plugin with very strict settings for Scala 2.12 and 2.13 projects
```sbt
addSbtPlugin("com.evolution" % "sbt-scalac-opts-plugin" % "<latest version>")
```

## Contributing

Contributors from outside the Evolution organization have to sign the
[Contributor License Agreement](CLA.md) before a pull request can be merged.
CLA Assistant posts a comment with the link on the pull request, and signing is
a one-time click that covers every later contribution.

## Revision numbers of known versions

It is suggested to use the very latest version of all workflows.
Revision numbers of older versions are provided only for reference purposes!

| Version | Revision number                            |
|---------|--------------------------------------------|
| v7.0.0  | `f02d2d572f3070fba1fd22edfb02070e57a0651b` |
| v6.4.0  | `2a50f1819b6fef2657ba802fd62612b7fc8450f0` |
| v5.1.0  | `7803a53309369dbada16835a5414af328fe76d8c` |
