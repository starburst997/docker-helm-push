# Docker Helm Push Action - Development Guide

## Project Overview
A GitHub Action that builds and pushes Docker images with Helm charts to container registries. Designed for CI/CD pipelines with Kubernetes deployments.

## Key Features
- Multi-platform Docker builds with customizable build arguments
- GitHub Actions cache integration for faster builds
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
- **Logic**: Parses semantic versions with regex, preserves suffixes and 'v' prefix behavior
- **Docker Tags**:
  - Input `v1.2.3-dev` → [v1.2.3-dev, v1.2-dev, v1-dev] (preserves 'v')
  - Input `1.2.3-dev` → [1.2.3-dev, 1.2-dev, 1-dev] (no 'v' added)
- **Helm Charts**: Only uses the original version (no breakdown), strips 'v' prefix (v1.2.3 → 1.2.3)

### 4. Additional Tags
- **Input**: `additional-tags` (comma-separated string)
- **Purpose**: Add extra tags like "latest" or "stable" independently of version
- **Processing**: Parsed and added to Docker tags (not Helm versions)

### 5. Docker Build Caching
- **Input**: `cache` (boolean, defaults to `true`)
- **Purpose**: Enable/disable GitHub Actions cache for Docker layer caching
- **Implementation**: Uses `type=gha` cache backend with `mode=max` for comprehensive layer caching
- **Benefits**: Significantly reduces build times for subsequent builds by reusing unchanged layers
- **Cache Scope**: Cache is scoped to the repository and branch, with automatic cleanup by GitHub Actions
- **Disable When**: Set to `false` for clean builds or when cache invalidation is needed

## Testing Scenarios

### Critical Test Cases
1. **Version Parsing**: Test with v1.2.3, 1.2.3, v1.2.3-dev, main-abc123
2. **Registry Auth**: Verify default github.token and custom PAT support
3. **Build Arguments**: Test mixing static values, secrets, and shell commands
4. **Multi-platform**: Ensure linux/amd64,linux/arm64 builds work
5. **Helm Charts**: Test with missing chart, wrong path, version mismatch
6. **Build Caching**: Verify cache hits on subsequent builds, test with cache disabled

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
- Build attestations and SBOM generation
- Custom Helm chart name (separate from image-name)
- Advanced cache configuration (custom cache backends, cache scopes)

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

## Documentation Requirements

**CRITICAL**: Whenever you add, modify, or remove an input or output in `action.yml`, you **MUST** update ALL of the following documentation files:

### Required Documentation Updates

1. **action.yml** - The source of truth
   - Add/modify/remove the input/output definition
   - Include clear description
   - Specify default value and required status

2. **README.md** - Main documentation
   - Update the Inputs table (around line 73)
   - Add dedicated section if it's a major feature
   - Update usage examples if relevant

3. **docs/index.html** - Website documentation
   - Update the Reference section (starting around line 783)
   - Add to appropriate reference card (Required Inputs, Optional Inputs, Docker Configuration, Helm Configuration)
   - Maintain consistent formatting with existing entries

4. **CLAUDE.md** - Development guide (this file)
   - Add to "Key Design Decisions" section if it's a significant feature
   - Update "Testing Scenarios" if new test cases are needed
   - Document implementation details and rationale

### Documentation Checklist

Before considering any input/output change complete, verify:

- [ ] Added to `action.yml` with description and default
- [ ] Added to README.md inputs table
- [ ] Added dedicated section in README.md if major feature
- [ ] Added to docs/index.html reference cards
- [ ] Added to CLAUDE.md Key Design Decisions (if significant)
- [ ] Added to CLAUDE.md Testing Scenarios (if needed)
- [ ] Updated any relevant usage examples

### Example: Adding a New Input

```yaml
# 1. In action.yml
inputs:
  new-feature:
    description: "Enable the new awesome feature"
    required: false
    default: "true"
```

```markdown
# 2. In README.md Inputs table
| `new-feature` | Enable the new awesome feature | No | `true` |

# 3. In README.md (add section if major)
## New Awesome Feature

Description of how it works...
```

```html
<!-- 4. In docs/index.html -->
<div class="param-item">
  <div class="param-name">new-feature</div>
  <div class="param-desc">Enable the new awesome feature</div>
  <div class="param-default">Default: true</div>
</div>
```

```markdown
# 5. In CLAUDE.md Key Design Decisions
### N. New Awesome Feature
- **Input**: `new-feature` (boolean, defaults to `true`)
- **Purpose**: Description of why this exists
- **Implementation**: Technical details
```

**Failure to update all documentation locations will result in incomplete work and user confusion.**