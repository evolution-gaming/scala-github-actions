# Scala GitHub Actions

## Scala CI workflow

Runs tests, coverage, binary compatibility, formatting and scaladoc on every push and pull request.
Replaces the hand-written `ci.yml` that each project used to carry.

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
    uses: evolution-gaming/scala-github-actions/.github/workflows/ci.yml@v6
    secrets: inherit
```

Nothing else is required if the project uses the defaults below. Coverage is uploaded to Coveralls,
so `secrets: inherit` is needed for `GITHUB_TOKEN`.

### Inputs

| input | default | notes |
|---|---|---|
| `scala_versions` | `'["2.13.18", "3.3.8"]'` | JSON array; becomes the build matrix |
| `java_version` | `'17'` | |
| `java_distribution` | `'temurin'` | |
| `test_task` | auto | `testFull` on sbt 2, `test` on sbt 1, read from `project/build.properties` |
| `coverage` | `true` | collect coverage and upload to Coveralls |
| `version_policy_check` | `true` | requires [sbt-version-policy](https://github.com/scalacenter/sbt-version-policy/) |
| `scalafmt_check` | `true` | |
| `doc_check` | `true` | runs `Compile/doc` |

Example for a project without `sbt-version-policy` and on a different Scala set:

```yaml
jobs:
  test:
    uses: evolution-gaming/scala-github-actions/.github/workflows/ci.yml@v6
    secrets: inherit
    with:
      scala_versions: '["2.13.18", "3.3.7"]'
      version_policy_check: false
```

### Why the steps are ordered this way

Two sbt 2 behaviours make a naive coverage setup report nothing while still passing:

* sbt 2's compile cache is **not** keyed on scoverage's instrumentation. If a plain compile runs
  first, the coverage build reuses those uninstrumented classes and the report comes out empty. The
  coverage build therefore runs **before** the formatting, binary-compatibility and scaladoc checks.
* `sbt/setup-sbt` restores `~/.cache/sbt` across runs by default, which reintroduces the same problem
  on any run whose build files did not change. This workflow sets `disk-cache: false` whenever
  coverage is enabled.

The workflow also fails if the produced cobertura report has no valid lines, so a silently empty
report is an error rather than a green build.

Binary compatibility, formatting and scaladoc run as **explicit sbt tasks**, not via a project-local
`check` alias. An alias can be stubbed out (`addCommandAlias("check", "show version")`), which makes
the gate silently guarantee nothing.

## Scala Release workflow (v3, v4, v5)

### Setup

To use Scala Release workflow have to set up project:
* add latest [sbt-dynver](https://github.com/sbt/sbt-dynver) plugin
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
