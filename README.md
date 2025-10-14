# Docker Helm Push Action

Build a Docker image and its corresponding Helm chart, then push both to a container registry — all in one simple GitHub Action. Perfect for Kubernetes deployments where you need both artifacts published together.

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

| Input               | Description                                | Required | Default                          |
| ------------------- | ------------------------------------------ | -------- | -------------------------------- |
| `registry`          | Container registry URL                     | No       | `ghcr.io`                        |
| `username`          | Username or organization                   | No       | `${{ github.repository_owner }}` |
| `image-name`        | Name of the Docker image                   | **Yes**  | -                                |
| `version`           | Version tag (e.g., v1.2.3, v1.2.3-dev)     | **Yes**  | -                                |
| `additional-tags`   | Additional tags to apply (comma-separated) | No       | `''`                             |
| `dockerfile`        | Path to the Dockerfile                     | No       | `./Dockerfile`                   |
| `context`           | Build context path                         | No       | `./`                             |
| `platforms`         | Target platforms for build                 | No       | `linux/amd64`                    |
| `helm-chart-path`   | Path to Helm chart directory               | No       | `helm`                           |
| `push-helm`         | Whether to push Helm chart                 | No       | `true`                           |
| `build-args`        | JSON array of build arguments and secrets  | No       | `[]`                             |
| `version-breakdown` | Enable semantic version breakdown          | No       | `false`                          |
| `github-token`      | GitHub token for authentication            | No       | `${{ github.token }}`            |

## Version Breakdown Feature

When `version-breakdown` is set to `true`, the action automatically creates multiple version tags for Docker images:

### Example: Version `v1.2.3` with `additional-tags: "latest,stable"`

- Docker tags: `v1.2.3`, `v1.2`, `v1`, `latest`, `stable`
- Helm chart version: `1.2.3`

### Example: Version `1.2.3-dev` (no 'v' prefix)

- Docker tags: `1.2.3-dev`, `1.2-dev`, `1-dev`
- Helm chart version: `1.2.3-dev`

**Note:** Helm charts always use a single semantic version without the 'v' prefix, following Helm's versioning standards. Only Docker images get multiple tags.

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
          helm-chart-path: helm
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
          # github-token: ${{ secrets.PAT_WITH_MORE_PERMISSIONS }}
```

## Helm Chart Structure

Your Helm chart should follow this structure:

```
helm/
└── my-app/
    ├── Chart.yaml
    ├── values.yaml
    ├── templates/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── ingress.yaml
    └── charts/
```

The action will:

1. Update Helm dependencies
2. Package the chart with the specified version
3. Push to the OCI registry at `oci://[registry]/[owner]/charts`

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
  github-token: ${{ secrets.DOCKER_HUB_TOKEN }}
```

## Troubleshooting

### Build Arguments Not Working

Ensure secrets are properly set in your repository settings and exposed as environment variables:

```yaml
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}
```

### Helm Push Failures

1. Verify the Helm chart exists at the specified path
2. Ensure the chart name matches the `image-name` input
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

## Support

For issues, feature requests, or questions, please open an issue in the GitHub repository
