# Docker Helm Push Action

Flexibly build Docker images and/or package Helm charts, then push them to a container registry — all in one simple GitHub Action. Perfect for Kubernetes deployments, supporting Docker-only, Helm-only, or combined workflows.

## Quick Start

### Basic Usage

```yaml
name: Build and Push

on:
  push:
    tags:
      - "v*"

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Build and Push Docker with Helm
        uses: starburst997/docker-helm-push@v1
        with:
          image-name: my-app
          version: ${{ github.ref_name }}
```

### Advanced Usage with Build Arguments

```yaml
name: Build and Push with Secrets

on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Build and Push Docker with Helm
        uses: starburst997/docker-helm-push@v1
        with:
          image-name: my-app
          version: ${{ github.ref_name }}
          additional-tags: latest,stable
          version-breakdown: true
          build-args: |
            [
              "NODE_ENV=production",
              "API_URL=https://api.example.com",
              "AUTH_APPLE_ID=${{ secrets.AUTH_APPLE_ID }}",
              "AUTH_APPLE_SECRET=${{ secrets.AUTH_APPLE_SECRET }}",
              "AUTH_APPLE_BUNDLE=${{ secrets.AUTH_APPLE_BUNDLE }}"
            ]
```

## Inputs

| Input               | Description                                | Required | Default                               |
| ------------------- | ------------------------------------------ | -------- | ------------------------------------- |
| `registry`          | Container registry URL                     | No       | `ghcr.io`                             |
| `username`          | Username or organization                   | No       | `${{ github.repository_owner }}`      |
| `image-name`        | Name of the Docker image                   | No       | `${{ github.event.repository.name }}` |
| `version`           | Version tag (e.g., v1.2.3, v1.2.3-dev)     | **Yes**  | -                                     |
| `additional-tags`   | Additional tags to apply (comma-separated) | No       | `latest`                              |
| `dockerfile`        | Path to the Dockerfile                     | No       | `./Dockerfile`                        |
| `context`           | Build context path                         | No       | `./`                                  |
| `platforms`         | Target platforms for build                 | No       | `linux/amd64`                         |
| `helm-chart-path`   | Path to Helm charts directory              | No       | `charts`                              |
| `push-helm`         | Whether to push Helm charts                | No       | `true`                                |
| `helm-strip-suffix` | Strip version suffix for Helm charts       | No       | `true`                                |
| `helm-namespace`    | Namespace for Helm charts in registry      | No       | `charts`                              |
| `build-args`        | JSON array of build arguments and secrets  | No       | `[]`                                  |
| `version-breakdown` | Enable semantic version breakdown          | No       | `true`                                |
| `cache`             | Enable Docker build caching                | No       | `true`                                |
| `token`             | GitHub token for authentication            | No       | `${{ github.token }}`                 |
| `git-push`          | Push commits and tags to remote            | No       | `false`                               |
| `make-public`       | Make packages public (ghcr.io only)        | No       | `false`                               |

## Version Breakdown Feature

When `version-breakdown` is set to `true`, the action automatically creates multiple version tags for Docker images:

### Example: Version `v1.2.3` with `additional-tags: "latest,stable"`

- Docker tags: `v1.2.3`, `v1.2`, `v1`, `latest`, `stable`
- Helm chart version: `1.2.3` (with `helm-strip-suffix: true`)

### Example: Version `1.2.3-dev` (no 'v' prefix)

- Docker tags: `1.2.3-dev`, `1.2-dev`, `1-dev`
- Helm chart version: `1.2.3` (suffix stripped with `helm-strip-suffix: true`)
- Or `1.2.3-dev` (if `helm-strip-suffix: false`)

**Note:** Helm charts always use a single semantic version without the 'v' prefix, following Helm's versioning standards. Only Docker images get multiple tags.

## Build Caching

The action includes intelligent caching to speed up both Docker and Helm builds by reusing unchanged artifacts.

### Docker Layer Caching

When `cache: true` (default), the action uses GitHub Actions cache to store Docker build layers. Subsequent builds on the same branch will reuse cached layers, significantly reducing build times.

```yaml
with:
  cache: true # Default, can be omitted
```

### Helm Dependencies Caching

The same `cache` input also controls Helm dependency caching. When enabled, Helm chart dependencies are cached to avoid repeated downloads.

**What gets cached:**

- Downloaded chart dependencies from Helm repositories
- Chart metadata and index files
- Dependency lock files

**Cache invalidation:**

- Automatically invalidates when `Chart.yaml` or `Chart.lock` files change
- Uses OS-specific cache keys for compatibility

### Disabling Cache

For clean builds or when you need to invalidate both Docker and Helm caches:

```yaml
with:
  cache: false # Force clean build without any caching
```

### Cache Behavior

- **Automatic**: No additional setup required
- **Branch-scoped**: Cache is isolated per branch
- **Smart invalidation**: Only rebuilds/redownloads when files change
- **GitHub-managed**: Automatic cleanup and lifecycle management
- **Unified control**: Single `cache` input controls both Docker and Helm caching

### Performance Impact

Typical improvements with caching enabled:

**Docker builds:**

- **First build**: Normal build time (cache population)
- **Subsequent builds**: 50-90% faster (depending on changes)
- **No code changes**: Near-instant builds

**Helm packaging:**

- **First build**: Normal dependency download time
- **Subsequent builds**: 70-95% faster (skips downloads)
- **No Chart.yaml changes**: Instant dependency resolution

## Build Arguments

Pass build arguments and secrets as a JSON array:

```yaml
build-args: |
  [
    "NODE_ENV=production",
    "API_URL=https://api.example.com",
    "NPM_TOKEN=${{ secrets.NPM_TOKEN }}",
    "DATABASE_URL=${{ secrets.DATABASE_URL }}"
  ]
```

Build arguments can include:

- Static values: `"NODE_ENV=production"`
- Environment variables: `"BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')"`
- GitHub context: `"VERSION=${{ github.sha }}"`
- Secrets: `"NPM_TOKEN=${{ secrets.NPM_TOKEN }}"`

## Complete Example Workflow

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    tags: ["v*"]
  pull_request:
    branches: [main]

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Determine Version
        id: version
        run: |
          if [[ "${{ github.ref }}" == refs/tags/v* ]]; then
            # Production release
            echo "version=${{ github.ref_name }}" >> $GITHUB_OUTPUT
            echo "additional_tags=latest,stable" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == refs/heads/main ]]; then
            # Main branch build
            SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
            echo "version=main-${SHORT_SHA}" >> $GITHUB_OUTPUT
            echo "additional_tags=latest" >> $GITHUB_OUTPUT
          else
            # Development build
            SHORT_SHA=$(echo "${{ github.sha }}" | cut -c1-7)
            echo "version=dev-${SHORT_SHA}" >> $GITHUB_OUTPUT
            echo "additional_tags=dev" >> $GITHUB_OUTPUT
          fi

      - name: Build and Push Docker with Helm
        uses: starburst997/docker-helm-push@v1
        with:
          registry: ghcr.io
          image-name: my-application
          version: ${{ steps.version.outputs.version }}
          additional-tags: ${{ steps.version.outputs.additional_tags }}
          dockerfile: ./Dockerfile
          context: ./
          platforms: linux/amd64,linux/arm64
          helm-chart-path: charts
          push-helm: true
          version-breakdown: ${{ startsWith(github.ref, 'refs/tags/v') }}
          build-args: |
            [
              "BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')",
              "VCS_REF=${{ github.sha }}",
              "VERSION=${{ steps.version.outputs.version }}",
              "NPM_TOKEN=${{ secrets.NPM_TOKEN }}",
              "SENTRY_AUTH_TOKEN=${{ secrets.SENTRY_AUTH_TOKEN }}"
            ]
          # Optional: Override default github.token if needed
          # token: ${{ secrets.PAT_WITH_MORE_PERMISSIONS }}
```

## Flexible Operation Modes

The action automatically detects what to build based on available files:

- **Docker only**: If Dockerfile exists but no Helm charts directory
- **Helm only**: If Helm charts exist but no Dockerfile
- **Both**: If both Dockerfile and Helm charts exist
- **Skip**: Gracefully skips missing components without failing

## Helm Chart Structure

The action supports both **single** and **multiple** Helm charts:

### Single Chart

When `Chart.yaml` exists directly in the `helm-chart-path` directory:

```
charts/
├── Chart.yaml
├── values.yaml
└── templates/
```

The chart will be published using the repository name (from `image-name` input).

### Multiple Charts

When multiple subdirectories contain charts:

```
charts/
├── app-frontend/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── app-backend/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── app-worker/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

The action will:

1. Automatically discover all charts in subdirectories
2. Update dependencies for each chart
3. Package each chart with the specified version
4. Push all charts to the OCI registry at `oci://[registry]/[username]/[helm-namespace]`

## Container Registry Authentication

### GitHub Container Registry (ghcr.io)

The default configuration uses GitHub Container Registry. Ensure your workflow has the necessary permissions:

```yaml
permissions:
  contents: read
  packages: write
```

### Other Registries

For other registries, specify the `registry` input and ensure proper authentication:

```yaml
with:
  registry: docker.io
  image-name: my-org/my-app
  token: ${{ secrets.DOCKER_HUB_TOKEN }}
```

## Troubleshooting

### Build Arguments Not Working

Ensure secrets are properly set in your repository settings and exposed as environment variables:

```yaml
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}
```

### Helm Push Failures

1. Verify at least one Helm chart exists in the specified directory
2. Ensure each chart has a valid Chart.yaml
3. Check registry permissions for chart uploads

### Version Breakdown Not Working

The version must follow semantic versioning format (e.g., `v1.2.3` or `1.2.3`). The action supports optional prefixes (`v`) and suffixes (`-dev`, `-beta`, etc.).

## Security Considerations

- Never hardcode sensitive values directly in `build-args`
- Always use GitHub secrets for sensitive data: `"TOKEN=${{ secrets.TOKEN }}"`
- Ensure proper repository permissions
- Regularly rotate access tokens
- Review build logs for accidental secret exposure

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Making Packages Public

When using GitHub Container Registry, you can automatically make packages public:

```yaml
with:
  make-public: true # Makes both Docker images and Helm charts public
```

This is useful for open-source projects where packages should be publicly accessible.

## Support

For issues, feature requests, or questions, please open an issue in the GitHub repository
