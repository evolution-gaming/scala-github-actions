# Scala GitHub Actions

## Scala Release workflow (v2)

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
        uses: evolution-gaming/scala-github-actions/.github/workflows/release.yml@v2
        secrets: inherit
    ```

### Usage

In project's repo:
* push tag as separate activity 
* crate new version tag, like `git tag v1.2.3 -a -m "release v1.2.3"`
* push the tag, `git push origin tag v1.2.3`

The above sequence will start Release workflow, which will:
* run SBT commands `+clean; +check; +all test package` to make sure that code quality is good
* run SBT command `+publish` to publish packaged artifacts
* if any of above steps will fail, the workflow will remove git tag - improve code and push fix, later tag again
* workflow will auto-generate release notes and will publish them
* go to `Code` and navigate to `Releases`
* review release notes and amend, if required

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
