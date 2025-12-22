# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository provides **reusable GitHub Actions workflows for Maven/Java projects** following the **Gitflow branching model**. It includes automated workflows for releases and hotfixes, with integrated Maven Central publishing via JReleaser.

**Key Features:**
- Gitflow-based release and hotfix workflows
- Self-contained Maven Central publishing (no external dependencies)
- Fully parameterizable and reusable via `workflow_call`
- Local composite actions for version management and publishing
- Support for manual fallback workflows

## Repository Structure

```
.github/
├── workflows/                  # Reusable GitHub Actions workflows
│   ├── release-start.yml      # Create release branch from main
│   ├── release-finish.yml     # Tag, publish, update main (REUSABLE)
│   ├── hotfix-start.yml       # Create hotfix branch from tag
│   ├── hotfix-finish.yml      # Tag, publish, cherry-pick to main (REUSABLE)
│   ├── release-manual.yml     # Manual release fallback
│   └── hotfix-manual.yml      # Manual hotfix fallback
└── actions/                    # Composite actions
    └── jreleaser-release/     # Handles JReleaser config and Maven Central publishing
```

## Gitflow Workflow Process

### Release Process

1. **Start Release** (`release-start.yml`):
   - Creates `release/X.Y.Z` branch from `main`
   - Sets version in pom.xml to `X.Y.Z-SNAPSHOT`

2. **Finish Release** (`release-finish.yml`):
   - Removes `-SNAPSHOT` from pom.xml
   - Creates git tag (e.g., `v2.0.16`)
   - Publishes to Maven Central
   - Merges release branch back to `main`
   - Updates `main` to next development version (e.g., `2.1.0-SNAPSHOT`)
   - Deletes release branch

### Hotfix Process

1. **Start Hotfix** (`hotfix-start.yml`):
   - Creates `hotfix/X.Y.Z.N` branch from a tag
   - Sets version in pom.xml to `X.Y.Z.N-SNAPSHOT`

2. **Finish Hotfix** (`hotfix-finish.yml`):
   - Removes `-SNAPSHOT` from pom.xml
   - Creates git tag (e.g., `v2.0.16.1`)
   - Publishes to Maven Central
   - Cherry-picks commits to `main` (optional)
   - Deletes hotfix branch

## Using as Reusable Workflows

The `release-finish.yml` and `hotfix-finish.yml` workflows can be called from other repositories.

### Example: Call Release Finish Workflow

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      release_branch:
        description: 'Release branch (e.g., release/2.0.16)'
        required: true

jobs:
  release:
    uses: your-org/ror-gha-workflows/.github/workflows/release-finish.yml@main
    with:
      release_branch: ${{ github.event.inputs.release_branch }}
      next_version_increment: 'minor'  # or 'major', 'patch'
      base_branch: 'main'  # or 'master', 'develop', etc.
      runner: 'ubuntu-24.04'
      java_version: 21
      java_distribution: 'liberica'
      version_tag_prefix: 'v'
      artifact_group_id: 'com.example'
      artifact_ids: 'my-library,my-cli'
    secrets:
      SONATYPE_AUTH_USER: ${{ secrets.SONATYPE_AUTH_USER }}
      SONATYPE_AUTH_TOKEN: ${{ secrets.SONATYPE_AUTH_TOKEN }}
      SONATYPE_GPG_KEY_PUBLIC: ${{ secrets.SONATYPE_GPG_KEY_PUBLIC }}
      SONATYPE_GPG_KEY: ${{ secrets.SONATYPE_GPG_KEY }}
      SONATYPE_GPG_KEY_PASSWORD: ${{ secrets.SONATYPE_GPG_KEY_PASSWORD }}
```

### Example: Call Hotfix Finish Workflow

```yaml
name: Hotfix
on:
  workflow_dispatch:
    inputs:
      hotfix_branch:
        description: 'Hotfix branch (e.g., hotfix/2.0.16.1)'
        required: true

jobs:
  hotfix:
    uses: your-org/ror-gha-workflows/.github/workflows/hotfix-finish.yml@main
    with:
      hotfix_branch: ${{ github.event.inputs.hotfix_branch }}
      merge_to_main: true
      base_branch: 'main'  # or 'master', 'develop', etc.
      runner: 'ubuntu-24.04'
      java_version: 21
      java_distribution: 'liberica'
      version_tag_prefix: 'v'
      artifact_group_id: 'com.example'
      artifact_ids: 'my-library,my-cli'
    secrets:
      SONATYPE_AUTH_USER: ${{ secrets.SONATYPE_AUTH_USER }}
      SONATYPE_AUTH_TOKEN: ${{ secrets.SONATYPE_AUTH_TOKEN }}
      SONATYPE_GPG_KEY_PUBLIC: ${{ secrets.SONATYPE_GPG_KEY_PUBLIC }}
      SONATYPE_GPG_KEY: ${{ secrets.SONATYPE_GPG_KEY }}
      SONATYPE_GPG_KEY_PASSWORD: ${{ secrets.SONATYPE_GPG_KEY_PASSWORD }}
```

## Workflow Parameters

### Common Parameters (release-finish.yml & hotfix-finish.yml)

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `runner` | string | No | `ubuntu-24.04` | GitHub runner to use |
| `java_version` | number | No | `21` | Java version for builds |
| `java_distribution` | string | No | `liberica` | Java distribution (liberica, temurin, etc.) |
| `version_tag_prefix` | string | No | `v` | Prefix for git tags (e.g., "v" creates "v2.0.16") |
| `base_branch` | string | No | `main` | Base development branch (main, master, develop, etc.) |
| `artifact_group_id` | string | No | `""` | Maven group ID for summary links |
| `artifact_ids` | string | No | `""` | Comma-separated artifact IDs for summary links |

### Release-Specific Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `release_branch` | string | Yes | - | Release branch to finish (e.g., `release/2.0.16`) |
| `next_version_increment` | string | No | `minor` | Next version type: major, minor, or patch |
| `next_version` | string | No | `""` | Or specify exact next version (e.g., `3.0.0-SNAPSHOT`) |

### Hotfix-Specific Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `hotfix_branch` | string | Yes | - | Hotfix branch to finish (e.g., `hotfix/2.0.16.1`) |
| `merge_to_main` | boolean | No | `true` | Cherry-pick hotfix commits to base branch |

## Required Secrets

All workflows require these secrets for Maven Central publishing:

| Secret | Description |
|--------|-------------|
| `SONATYPE_AUTH_USER` | Sonatype Central username |
| `SONATYPE_AUTH_TOKEN` | Sonatype Central password/token |
| `SONATYPE_GPG_KEY_PUBLIC` | Base64-encoded GPG public key |
| `SONATYPE_GPG_KEY` | Base64-encoded GPG secret key |
| `SONATYPE_GPG_KEY_PASSWORD` | GPG key passphrase |

### Setting up Secrets

1. **Generate GPG Keys** (if you don't have them):
   ```bash
   gpg --full-generate-key
   # Follow prompts, use RSA, 4096 bits
   ```

2. **Export and Encode Keys**:
   ```bash
   # Export public key (base64)
   gpg --armor --export YOUR_EMAIL | base64 -w 0

   # Export secret key (base64)
   gpg --armor --export-secret-keys YOUR_EMAIL | base64 -w 0
   ```

3. **Add to GitHub**:
   - Go to repository Settings → Secrets and variables → Actions
   - Add each secret with the exact names above

## JReleaser Configuration

The workflows use the local `.github/actions/jreleaser-release` composite action, which:

1. Generates JReleaser configuration dynamically
2. Stages artifacts to Maven Central
3. Performs full release to Maven Central

**Environment Variables Set:**
- `JRELEASER_MAVENCENTRAL_URL`: Maven Central publisher API
- `JRELEASER_DEPLOY_MAVEN_MAVENCENTRAL_ACTIVE`: RELEASE
- `JRELEASER_NEXUS2_URL`: Nexus staging API
- `JRELEASER_OVERWRITE`: true
- `JRELEASER_UPDATE`: true
- `JRELEASER_GIT_ROOT_SEARCH`: true

## Troubleshooting

### Publishing Fails with "Unauthorized"

**Problem:** JReleaser can't authenticate with Maven Central.

**Solution:**
- Verify `SONATYPE_AUTH_USER` and `SONATYPE_AUTH_TOKEN` are correct
- Ensure you have publishing rights for the group ID
- Check if token has expired

### Tag Already Exists

**Problem:** Workflow fails because git tag already exists.

**Solution:**
- Delete the tag: `git tag -d v2.0.16 && git push origin :refs/tags/v2.0.16`
- Or use manual workflow to republish existing tag

### GPG Signing Fails

**Problem:** JReleaser can't sign artifacts.

**Solution:**
- Verify GPG keys are properly base64-encoded
- Ensure `SONATYPE_GPG_KEY_PASSWORD` is correct
- Check that secret key contains both public and private parts

### Cherry-pick Conflicts in Hotfix

**Problem:** Merge to main fails with conflicts.

**Solution:**
- Set `merge_to_main: false` to skip automatic merge
- Manually cherry-pick commits after hotfix is published
- Resolve conflicts locally and push to main

### Version Mismatch

**Problem:** Tag version doesn't match pom.xml version.

**Solution:**
- Ensure `release-start` or `hotfix-start` workflows completed successfully
- Verify pom.xml was updated correctly before finishing
- Check that `-SNAPSHOT` was properly added/removed

## Maven Project Requirements

For these workflows to work, your Maven project must have:

1. **JReleaser Maven Plugin** configured in `pom.xml`:
   ```xml
   <build>
     <plugins>
       <plugin>
         <groupId>org.jreleaser</groupId>
         <artifactId>jreleaser-maven-plugin</artifactId>
         <version>1.x.x</version>
       </plugin>
     </plugins>
   </build>
   ```

2. **Publication Profile** for Maven Central deployment

3. **Multi-module support** (if applicable):
   - The workflows use `-DprocessAllModules=true` to update all modules

## Best Practices

1. **Always use start/finish pairs**: Run `release-start` before `release-finish`
2. **Test before releasing**: Build and test on release/hotfix branch before finishing
3. **Use semantic versioning**: Follow major.minor.patch for releases, major.minor.patch.hotfix for hotfixes
4. **Backup before manual workflows**: Manual workflows are for emergencies only
5. **Verify artifact links**: Provide `artifact_group_id` and `artifact_ids` for better summaries
6. **Keep secrets updated**: Rotate tokens and keys periodically

## Local Development

When testing workflows locally, you can use:

1. **Act** (https://github.com/nektos/act) to run GitHub Actions locally
2. **Workflow Dispatch** to manually trigger workflows with test parameters
3. **Draft releases** for testing before actual release

## Support and Maintenance

### Updating Workflows

When modifying workflows:
1. Test with `workflow_dispatch` in a test repository first
2. Verify all parameterization works correctly
3. Update this documentation to reflect changes
4. Tag a new version of this repository for stable references

### Version Compatibility

- **GitHub Actions**: runner-images ubuntu-24.04+
- **Java**: 17+ (default: 21)
- **Maven**: 3.6+ (uses wrapper if available)
- **JReleaser**: 1.x (configured in project pom.xml)