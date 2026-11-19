# CC Java SDK

This is the SDK implementation for authoring Java cloud compute plugins.

## Building

Use Java 17 and the checked-in Gradle wrapper (no separate Gradle install needed):

```sh
./gradlew build             # Compile, test, and build the SDK JAR
./gradlew build -x test     # Build without running tests
./gradlew shadowJar        # Build a bundled JAR containing dependencies
```

Outputs are in `build/libs/`. The normal SDK JAR is for Maven/Gradle consumers;
the `-all.jar` is the bundled distribution.

## Automated releases

The workflow in `.github/workflows/release.yml` runs when a tag matching `v*`
is pushed. It validates that the tag is `vMAJOR.MINOR.PATCH` (for example,
`v1.1.3`; prereleases and leading zeros are not supported), then:

1. Builds the SDK and bundled JAR using Java 17.
2. Publishes `usace.cc:cc-java-sdk:<version>` to
   [GitHub Packages](https://github.com/USACE-Cloud-Compute/cc-java-sdk/packages).
3. Creates a GitHub Release with generated notes and attaches the bundled JAR.

**Tests are temporarily skipped in the release workflow.** Compilation and
packaging failures still block publishing, but this is not yet a tested-release
gate. See the CI test follow-up below.

The tag, without its `v`, supplies the published version through Gradle's
`releaseVersion` property. No matching version edit in `build.gradle` is needed.
Local builds retain the default version in that file; to build a release version
locally, use `./gradlew build -x test -PreleaseVersion=1.1.3`.

### Publishing a release

Merge the workflow and desired SDK changes first, then tag the intended commit:

```sh
git tag -a v1.1.3 -m "Release v1.1.3"
git push origin v1.1.3
```

Use a new version for each release (`v1.1.2` already exists). Pushing this workflow
does not retroactively release existing tags. Do not move tags or overwrite
published versions. Tag the commit you intend to ship, and restrict release-tag
creation with repository rulesets where appropriate.

The workflow uses the automatic `GITHUB_TOKEN`; no publishing PAT is normally
needed. Repository/organization policy must allow Actions to write repository
contents and packages. If a package already exists, ensure this repository has
permission to publish to it.

Publishing happens before the GitHub Release is created, so failed builds do not
announce releases. These operations are not atomic: if publishing succeeds but
release creation fails, finish the GitHub Release manually rather than blindly
rerunning publication of the same version. Creating a release manually no longer
triggers this workflow.

### Consuming the SDK

Configure the Maven repository
`https://maven.pkg.github.com/usace-cloud-compute/cc-java-sdk` and depend on
`usace.cc:cc-java-sdk:<version>`. Maven dependency metadata supplies the SDK's
dependencies; consumers normally do not need the bundled JAR.

GitHub's Maven registry requires authentication even for public packages.
For local downloads, use a GitHub username and a classic PAT with `read:packages`
(plus any required organization authorization). Consumer Actions workflows can
use their `GITHUB_TOKEN` where package access permits it. Never commit tokens.

## CI test follow-up

The current tests assume a configured plugin environment rather than a clean
GitHub runner. The team should address these before enabling the release test gate:

- **`PluginManagerTest`, `PayloadTest`, and `PayloadAttributeTest`:** initialize
  `PluginManager`, which defaults to S3 payload loading. Supply deterministic local
  payload fixtures and explicitly set the required environment variables. The
  existing filesystem store may help avoid real AWS access.
- **`IOManagerTest`:** uses remote S3 objects and `/data/testfile`. Use a seeded,
  isolated S3-compatible test service or a dedicated test bucket, temporary local
  directories, and independent setup/cleanup for every test. Tests must not depend
  on execution order or production data.
- **`EventStoreTest`:** supply a disposable TileDB store and its required runtime
  configuration; assert the returned data rather than only printing it, and fail
  instead of silently passing when the expected store is missing.
- **`SubstituterTest`:** appears self-contained; keep this kind of test in the
  default unit-test suite.
- Separate external-service integration tests from unit tests using JUnit tags
  and/or a dedicated Gradle test task. Document service startup, fixture seeding,
  required environment variables, and cleanup. Keep credentials out of fixtures
  and never run tests that write against production storage.

Once ready, remove `-x test` from the workflow's build command (and its temporary
skip comment), run any required integration-test task before publication, and
upload `build/reports/tests/` and `build/test-results/` even when tests fail.
Consider also running the same checks on pull requests so failures are caught
before a release tag is pushed.
