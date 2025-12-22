# Reusable GitHub Actions Workflows for Maven Projects

Reusable GitHub Actions workflows for Maven/Java projects following the **Gitflow branching model**. Automates releases and hotfixes with integrated Maven Central publishing via JReleaser.

## Features

- **Gitflow-based workflows** for releases and hotfixes
- **Automated Maven Central publishing** with JReleaser
- **Fully parameterizable** and reusable via `workflow_call`
- **Self-contained** - no external action dependencies
- **Local composite actions** for version management
- **Manual fallback workflows** for edge cases
- **Configurable runners, Java versions, and distributions**

## Quick Start

### Using in Your Repository

Call these workflows from your repository to automate release and hotfix processes.

#### Release Workflow

Create `.github/workflows/release.yml` in your repository:

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      release_branch:
        description: 'Release branch (e.g., release/2.0.16)'
        required: true
      next_version_increment:
        description: 'Next version increment'
        required: true
        default: 'minor'
        type: choice
        options:
          - major
          - minor
          - patch

jobs:
  release:
    uses: your-org/ror-gha-workflows/.github/workflows/release-finish.yml@main
    with:
      release_branch: ${{ github.event.inputs.release_branch }}
      next_version_increment: ${{ github.event.inputs.next_version_increment }}
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

#### Hotfix Workflow

Create `.github/workflows/hotfix.yml` in your repository:

```yaml
name: Hotfix
on:
  workflow_dispatch:
    inputs:
      hotfix_branch:
        description: 'Hotfix branch (e.g., hotfix/2.0.16.1)'
        required: true
      merge_to_main:
        description: 'Cherry-pick commits to main'
        required: true
        default: true
        type: boolean

jobs:
  hotfix:
    uses: your-org/ror-gha-workflows/.github/workflows/hotfix-finish.yml@main
    with:
      hotfix_branch: ${{ github.event.inputs.hotfix_branch }}
      merge_to_main: ${{ github.event.inputs.merge_to_main }}
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

## How It Works

### Release Process

1. **Start Release**: Creates `release/X.Y.Z` branch from `main` with version `X.Y.Z-SNAPSHOT`
2. **Develop**: Make changes, test, and commit on release branch
3. **Finish Release**:
   - Removes `-SNAPSHOT` suffix
   - Creates git tag (e.g., `v2.0.16`)
   - Publishes to Maven Central
   - Merges release branch back to `main`
   - Updates `main` to next development version
   - Deletes release branch

### Hotfix Process

1. **Start Hotfix**: Creates `hotfix/X.Y.Z.N` branch from a production tag
2. **Fix**: Apply bug fixes and test on hotfix branch
3. **Finish Hotfix**:
   - Removes `-SNAPSHOT` suffix
   - Creates git tag (e.g., `v2.0.16.1`)
   - Publishes to Maven Central
   - Cherry-picks commits to `main` (optional)
   - Deletes hotfix branch

## Workflow Parameters

### Common Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `runner` | string | `ubuntu-24.04` | GitHub runner to use |
| `java_version` | number | `21` | Java version for builds |
| `java_distribution` | string | `liberica` | Java distribution (liberica, temurin, etc.) |
| `version_tag_prefix` | string | `v` | Prefix for git tags |
| `base_branch` | string | `main` | Base development branch (main, master, develop, etc.) |
| `artifact_group_id` | string | `""` | Maven group ID for summary links |
| `artifact_ids` | string | `""` | Comma-separated artifact IDs |

### Release-Specific

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `release_branch` | string | Yes | - | Release branch (e.g., `release/2.0.16`) |
| `next_version_increment` | string | No | `minor` | Next version: major, minor, or patch |
| `next_version` | string | No | `""` | Or specify exact version (e.g., `3.0.0-SNAPSHOT`) |

### Hotfix-Specific

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `hotfix_branch` | string | Yes | - | Hotfix branch (e.g., `hotfix/2.0.16.1`) |
| `merge_to_main` | boolean | No | `true` | Cherry-pick commits to base branch |

## Required Secrets

Set up these secrets in your repository (Settings → Secrets and variables → Actions):

| Secret | Description |
|--------|-------------|
| `SONATYPE_AUTH_USER` | Sonatype Central username |
| `SONATYPE_AUTH_TOKEN` | Sonatype Central password/token |
| `SONATYPE_GPG_KEY_PUBLIC` | Base64-encoded GPG public key |
| `SONATYPE_GPG_KEY` | Base64-encoded GPG secret key |
| `SONATYPE_GPG_KEY_PASSWORD` | GPG key passphrase |

### Setting up GPG Keys

```bash
# Generate GPG key (if needed)
gpg --full-generate-key

# Export public key (base64)
gpg --armor --export YOUR_EMAIL | base64 -w 0

# Export secret key (base64)
gpg --armor --export-secret-keys YOUR_EMAIL | base64 -w 0
```

## Maven Project Requirements

Your Maven project must have:

1. **JReleaser Maven Plugin** in `pom.xml`:
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

2. **Maven Central deployment configuration** in your publication profile

## Repository Structure

```
.github/
├── workflows/              # Reusable workflows
│   ├── release-start.yml
│   ├── release-finish.yml
│   ├── hotfix-start.yml
│   ├── hotfix-finish.yml
│   ├── release-manual.yml
│   └── hotfix-manual.yml
└── actions/                # Composite actions
    └── jreleaser-release/
```

## Documentation

For detailed documentation, see:

- [GITFLOW.md](.github/GITFLOW.md) - Gitflow branching model explained
- [RELEASE-PROCESS.md](.github/RELEASE-PROCESS.md) - Complete release process guide
- [CLAUDE.md](CLAUDE.md) - Technical reference for Claude Code

## Troubleshooting

### Publishing Fails

- Verify Sonatype credentials are correct
- Ensure you have publishing rights for the group ID
- Check if authentication token has expired

### Tag Already Exists

```bash
# Delete the tag locally and remotely
git tag -d v2.0.16
git push origin :refs/tags/v2.0.16
```

### GPG Signing Fails

- Verify GPG keys are properly base64-encoded
- Ensure `SONATYPE_GPG_KEY_PASSWORD` is correct
- Check that secret key contains both public and private parts

### Cherry-pick Conflicts

- Set `merge_to_main: false` to skip automatic merge
- Manually cherry-pick commits after hotfix is published
- Resolve conflicts locally and push to main

## Best Practices

1. Always use start/finish workflow pairs
2. Test thoroughly on release/hotfix branch before finishing
3. Follow semantic versioning (major.minor.patch)
4. Use hotfix format major.minor.patch.N for hotfixes
5. Keep secrets updated and rotate periodically
6. Provide `artifact_group_id` and `artifact_ids` for better workflow summaries

## Version Compatibility

- **GitHub Actions**: ubuntu-24.04+
- **Java**: 17+ (default: 21)
- **Maven**: 3.6+
- **JReleaser**: 1.x

## License

[Add your license here]

## Contributing

[Add contribution guidelines here]

## Support

For issues or questions, please [open an issue](../../issues).