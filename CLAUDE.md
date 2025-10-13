# Docker Helm Push Action - Development Guide

## Project Overview
A GitHub Action that builds and pushes Docker images with Helm charts to container registries. Designed for CI/CD pipelines with Kubernetes deployments.

## Key Features
- Multi-platform Docker builds with customizable build arguments
- Automatic Helm chart packaging and OCI registry push
- Semantic version breakdown (v1.2.3 → v1.2, v1)
- Flexible registry support (defaults to ghcr.io)
- Automatic GitHub token authentication

## Architecture

### Core Components
1. **action.yml**: Main action definition with inputs and composite steps
2. **Version Parser**: Handles semantic versioning breakdown and tag generation
3. **Build Args Processor**: Converts JSON array to Docker build arguments
4. **Docker Build**: Uses buildx for multi-platform builds
5. **Helm Publisher**: Packages and pushes charts to OCI registries

### Input Processing Flow
```
Inputs → Version Parsing → Tag Generation → Docker Build → Helm Package → Registry Push
```

## Key Design Decisions

### 1. Simplified Secrets Handling
- **Decision**: Removed separate `build-secrets` input
- **Rationale**: Users can pass secrets directly in `build-args` using `${{ secrets.NAME }}`
- **Implementation**: All arguments processed uniformly through JSON array

### 2. Username Flexibility
- **Input**: `username` (defaults to `github.repository_owner`)
- **Purpose**: Allow pushing to different organizations or custom registry paths
- **Usage**: Affects both Docker image paths and Helm chart registry paths

### 3. Version Breakdown
- **Input**: `version-breakdown` (boolean)
- **Logic**: Parses semantic versions with regex, preserves suffixes
- **Example**: v1.2.3-dev → [v1.2.3-dev, v1.2-dev, v1-dev]

### 4. Additional Tags
- **Input**: `additional-tags` (comma-separated string)
- **Purpose**: Add extra tags like "latest" or "stable" independently of version
- **Processing**: Parsed and added to Docker tags (not Helm versions)

## Testing Scenarios

### Critical Test Cases
1. **Version Parsing**: Test with v1.2.3, 1.2.3, v1.2.3-dev, main-abc123
2. **Registry Auth**: Verify default github.token and custom PAT support
3. **Build Arguments**: Test mixing static values, secrets, and shell commands
4. **Multi-platform**: Ensure linux/amd64,linux/arm64 builds work
5. **Helm Charts**: Test with missing chart, wrong path, version mismatch

### Edge Cases to Handle
- Empty build-args array
- Missing Helm chart at specified path
- Non-semantic version strings
- Registry authentication failures
- Spaces in build argument values

## Common Issues & Solutions

### Build Args Not Applied
- Ensure proper JSON escaping in workflow
- Check for quotes around values with spaces
- Verify secrets are available in repository

### Helm Push Failures
- Chart name must match `image-name` input
- Verify OCI registry support for Helm charts
- Check registry permissions for chart namespace

### Version Breakdown Not Working
- Version must match semantic format
- Regex: `^v?([0-9]+)\.([0-9]+)\.([0-9]+)(-.*)?$`
- Suffix preserved across all generated versions

## Future Enhancements
- Support for custom tag templates
- Parallel builds for multiple platforms
- Cache layer optimization
- Build attestations and SBOM generation
- Custom Helm chart name (separate from image-name)

## Development Commands

### Local Testing
```bash
# Test action locally with act
act -P ubuntu-latest=ghcr.io/catthehacker/ubuntu:latest

# Validate action.yml
yamllint action.yml

# Test version parsing regex
echo "v1.2.3-dev" | grep -E '^v?([0-9]+)\.([0-9]+)\.([0-9]+)(-.*)?$'
```

### Publishing
```bash
# Create release tag
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0

# Move major version tag
git tag -fa v1 -m "Update v1 tag"
git push origin v1 --force
```

## Security Notes
- Never log secret values in action
- Use `::add-mask::` for sensitive outputs
- Default to github.token for least privilege
- Validate all user inputs before use
- Sanitize build arguments to prevent injection