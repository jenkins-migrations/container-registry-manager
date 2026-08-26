# Jenkins to GitHub Actions Migration Report

## Scope

The migration analyzed only these Jenkins sources:

- `Jenkinsfile`
- `kspluginparalleltesting/Jenkinsfile`
- `parallelbuilds/Jenkinsfile`

The originals are archived at matching paths under `.github/ci-archive/` and
were removed from their former locations.

## Created workflows

| Archived source | GitHub Actions workflow | Trigger |
| --- | --- | --- |
| `.github/ci-archive/Jenkinsfile` | `.github/workflows/enterprise-ci.yml` | Manual dispatch |
| `.github/ci-archive/kspluginparalleltesting/Jenkinsfile` | `.github/workflows/ksplugin-parallel-testing.yml` | Manual dispatch |
| `.github/ci-archive/parallelbuilds/Jenkinsfile` | `.github/workflows/parallel-builds.yml` | Manual dispatch |

Manual dispatch preserves the absence of an SCM or timer trigger in the source
pipelines and prevents these example pipelines from running unexpectedly.

## Behavior mapping

### Enterprise CI

- Preserves Java 8 and Maven 3.3.9, unit and
  integration tests, Cucumber JSON collection, Sonar analysis and quality-gate
  polling, development and acceptance approvals, sanity/E2E tests, master-only
  release tagging, Maven deployment, artifact retention, and triggering the two
  downstream Jenkins deployment jobs.
- `getDevVersion()` and `getReleaseVersion()` are expanded inline with shell and
  Maven expressions.
- Jenkins approvals map to the `development` and `acceptance` GitHub
  environments.
- Jenkins credentials used for tag pushes map to the job-scoped
  `GITHUB_TOKEN`; Maven repository credentials map to a settings-file secret.
- Ephemeral hosted runners replace `deleteDir()`.

### Kubernetes plugin parallel testing

- Preserves cancellation of the previous run, two fail-fast kind shards,
  Java 17, kind test execution, test and log artifacts, JaCoCo data renaming,
  class/coverage transfer, the JDK 17 build, and merged coverage generation.
- Jenkins `splitTests` is replaced by an inline deterministic two-way split of
  sorted `*Test.java` files.
- `infra.prepareToPublishIncrementals()` is expanded into Maven metadata lookup
  and upload of the matching local-repository artifacts.
- `infra.maybePublishIncrementals()` only publishes when running on Jenkins
  infrastructure. That condition is false on GitHub-hosted runners, so the
  migrated workflow preserves its skip behavior.

### Parallel builds

- The Jenkins `linux` and `windows` branches map to a two-entry matrix with
  platform-native shells and the same build/test commands.

## Actions and pinning

Every action is pinned to a full commit SHA:

- `actions/checkout` v7.0.1:
  `3d3c42e5aac5ba805825da76410c181273ba90b1`
- `actions/setup-java` v6.0.0:
  `dd06d9cba3e5552c54d9f8ea23572deb30010f7c`
- `actions/upload-artifact` v7.0.1:
  `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a`
- `actions/download-artifact` v8.0.1:
  `3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c`

## Required secrets

| Secret | Purpose |
| --- | --- |
| `SONAR_HOST_URL` | Base URL of the SonarQube server. |
| `SONAR_TOKEN` | Sonar analysis and quality-gate API token. |
| `MAVEN_SETTINGS_XML` | Complete Maven `settings.xml` containing repository/server credentials required by `deploy`. |
| `JENKINS_URL` | Base URL of the retained Jenkins controller hosting deployment jobs. |
| `JENKINS_USER` | Jenkins API user allowed to trigger deployment jobs. |
| `JENKINS_API_TOKEN` | API token for that Jenkins user. |

No repository or environment variables are required.

## Manual prerequisites

1. Create GitHub environments named `development` and `acceptance`; configure
   required reviewers to reproduce Jenkins input approvals.
2. Add the required secrets, preferably to only the environments/jobs that use
   them.
3. Grant Actions `Read and write permissions` for repository contents so the
   master release job can force-update its release tag.
4. Ensure the SonarQube server is reachable from GitHub-hosted runners.
5. Keep the Jenkins `ApplicationToDev` and `AApplicationToACC` jobs reachable
   until deployment is migrated, and permit token-authenticated
   `buildWithParameters` requests.
6. Ensure Maven Central/archive access and all deployment repositories are
   reachable from hosted runners.
7. Ensure the Kubernetes test repository supplies `kind.sh`, the Maven project,
   Docker/kind prerequisites, and the `merge-jacoco-reports` profile.

## Known gaps and differences

- Incrementals candidates are retained as a GitHub artifact, but they are not
  submitted to the Jenkins-only publication service. Add a GitHub-compatible
  publication endpoint before enabling Incrementals publication.
- Jenkins `splitTests` can use historical timing and file-change estimates.
  The replacement is deterministic but does not balance using historical
  duration data.
- Jenkins `build` normally waits for and propagates the downstream deployment
  result. The REST trigger confirms queue acceptance only; deployment
  completion/result polling must be added when the downstream Jenkins API and
  authentication behavior are known.
- The development Jenkins job was documented as fetching its JAR from the
  Jenkins workspace. GitHub-hosted runners do not share that workspace; the
  retained job must be updated to download the GitHub artifact (or the
  development artifact must be published to a shared repository) before this
  deployment path is functional.
- Jenkins approval timeouts (three minutes) have no direct GitHub environment
  equivalent. Job timeouts apply after a runner starts, not while approval is
  pending.
- Jenkins applied one 25-minute timeout to the complete pipeline. GitHub Actions
  has no workflow-level timeout, so equivalent limits are applied to the
  long-running build and release jobs, with shorter limits on test/deploy jobs.
- Jenkins email notifications and changelog formatting are replaced by native
  GitHub Actions run notifications; `EMAIL_RECIPIENTS` is not migrated.
- Jenkins Cucumber and JUnit publishers rendered dedicated UI reports. The
  migration retains raw report files as artifacts without adding unverified
  report-publishing actions.
- Jenkins coverage UI/source-retention behavior is replaced by an uploaded
  JaCoCo XML artifact.
- Jenkins retained the latest five builds; GitHub has no direct per-workflow
  run-count retention setting. Generated artifacts use a five-day retention
  period, while repository/org Actions retention controls run history.
- The Jenkins retry conditions targeted lost Kubernetes/nonresumable agents,
  not ordinary test failures. GitHub-hosted runner recovery differs and no
  broad test retry was added.

## Validation

- All three workflow files parsed successfully as YAML.
- All 24 `uses:` references were verified to use full 40-character commit SHAs.
- `git diff --check` passed.
- Each archived Jenkinsfile was verified byte-for-byte against its original.
- Secret scanning reported no secrets.
- CodeQL analysis of the Actions workflows reported no alerts.
- No existing `actionlint` executable was available, so actionlint validation
  was skipped rather than installing an unrequested tool.
