# Jenkins to GitHub Actions Migration Report

## Summary

Migrated the repository's Jenkins pipelines to GitHub Actions workflows and archived the original Jenkinsfiles for reference.

## Source Jenkins pipelines

| Original path | Pipeline type | Archived path | GitHub Actions workflow |
| --- | --- | --- | --- |
| `Jenkinsfile` | Declarative pipeline | `.github/ci-archive/Jenkinsfile` | `.github/workflows/enterprise-maven-ci-cd.yml` |
| `kspluginparalleltesting/Jenkinsfile` | Scripted pipeline | `.github/ci-archive/kspluginparalleltesting/Jenkinsfile` | `.github/workflows/kubernetes-plugin-parallel-testing.yml` |
| `parallelbuilds/Jenkinsfile` | Scripted pipeline | `.github/ci-archive/parallelbuilds/Jenkinsfile` | `.github/workflows/parallel-platform-builds.yml` |

## Workflow mapping

### Enterprise Maven CI/CD

- Jenkins `agent { label 'java8' }` maps to `ubuntu-latest` with Temurin Java 8.
- Maven package, integration test, SonarQube scan, quality gate, development deployment, sanity test, release, acceptance deployment, and E2E stages map to separate GitHub Actions jobs with `needs` dependencies.
- Jenkins build version helpers are expanded inline using shell and Python to derive short commit versions from `git rev-parse` and `pom.xml`.
- Jenkins `input` approvals map to GitHub Environments named `development` and `acceptance`; configure required reviewers on those environments to require manual approval.
- Jenkins JUnit, Cucumber, and archive steps map to `actions/upload-artifact` uploads.
- Jenkins email notifications are replaced by GitHub Actions run notifications.

### Kubernetes Plugin Parallel Testing

- Jenkins scripted `splitTests` logic is expanded inline into a two-shard matrix that creates Surefire include/exclude files.
- Jenkins parallel branches map to matrix and independent jobs.
- Jenkins `infra.prepareToPublishIncrementals()` and `infra.maybePublishIncrementals()` shared-library calls are expanded inline as Maven deploy steps gated by `PUBLISH_INCREMENTALS=true`.
- Jenkins stash/unstash and coverage archival are represented with workflow artifacts and a JaCoCo aggregation job.

### Parallel Platform Builds

- Jenkins dynamic `linux` and `windows` parallel nodes map to a matrix over `ubuntu-latest` and `windows-latest`.
- Jenkins `sh` and `bat` commands map to Bash and PowerShell steps.

## Required secrets and variables

| Name | Type | Used by | Notes |
| --- | --- | --- | --- |
| `SONAR_TOKEN` | Secret | Enterprise Maven CI/CD | Required to run SonarQube analysis and quality gate checks. |
| `SONAR_HOST_URL` | Repository/environment variable | Enterprise Maven CI/CD | SonarQube server URL. |
| `SONAR_PROJECT_KEY` | Repository/environment variable | Enterprise Maven CI/CD | Required for quality gate polling. |
| `GITHUB_TOKEN` | Automatic secret | Enterprise Maven CI/CD | Used by GitHub Actions for release tag pushes when `contents: write` is granted. |
| `MAVEN_SETTINGS_XML` | Secret | Enterprise Maven CI/CD, Kubernetes Plugin Parallel Testing | Maven `settings.xml` content required for release deploys and incrementals publication. |
| `PUBLISH_INCREMENTALS` | Repository/environment variable | Kubernetes Plugin Parallel Testing | Set to `true` to enable the expanded incrementals publication steps. |

## Action security

All marketplace actions are from verified GitHub-owned publishers and pinned to commit SHAs:

- `actions/checkout@11d5960a326750d5838078e36cf38b85af677262` (v4)
- `actions/setup-java@cf277c60eb25467037889841efdb72551f06f6c3` (v4)
- `actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02` (v4)
- `actions/download-artifact@d3f86a106a0bac45b974a628896c90dbdf5c8093` (v4)

## Validation

- Original Jenkinsfiles were moved to `.github/ci-archive/` and removed from their original paths.
- Shared-library and plugin-specific Jenkins behavior was expanded inline where practical, with deployment and incrementals publication exposed as explicit workflow steps.
- Run `actionlint` against `.github/workflows/*.yml` to validate workflow syntax.

## Follow-up configuration

1. Configure `development` and `acceptance` GitHub Environments with required reviewers to preserve Jenkins approval gates.
2. Add SonarQube variables/secrets if SonarQube scanning should run.
3. Add `MAVEN_SETTINGS_XML` with Maven/Nexus deployment credentials before enabling release or incrementals publication on protected branches.
4. Replace the placeholder development and acceptance deployment echo commands with the repository's actual deployment implementation.
