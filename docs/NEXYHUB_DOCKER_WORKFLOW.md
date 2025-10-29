# NexyHub Docker Build Workflow

This document explains how the NexyHub Docker build and publishing workflow works.

## Overview

The workflow automatically builds and publishes NexyHub-branded n8n Docker images to GitHub Container Registry (ghcr.io) whenever you push to the `nexyhub-branding` branch.

## Automatic Triggers

The workflow triggers automatically on:
- **Push to `nexyhub-branding` branch** (excluding documentation-only changes)
- Changes to theme files, source code, or dependencies

## Manual Trigger

You can also trigger the build manually:

1. Go to **Actions** tab in GitHub
2. Select **NexyHub: Build and Push Docker Image** workflow
3. Click **Run workflow**
4. Choose whether to push to registry (default: yes)

## What Gets Built

The workflow:
1. ✅ Builds n8n with your NexyHub theme compiled in
2. ✅ Creates Docker images for both AMD64 and ARM64 platforms
3. ✅ Pushes to GitHub Container Registry
4. ✅ Creates multi-arch manifest for easy pulling

## Published Images

After a successful build, the following images are available:

### Multi-Architecture Tags (Recommended)
```bash
# Latest NexyHub build (multi-arch)
ghcr.io/nexyhub/n8n:nexyhub-latest

# Branch-specific tag (multi-arch)
ghcr.io/nexyhub/n8n:nexyhub-branding

# Commit-specific tag (multi-arch)
ghcr.io/nexyhub/n8n:nexyhub-branding-<short-sha>
```

### Platform-Specific Tags
```bash
# AMD64 only
ghcr.io/nexyhub/n8n:nexyhub-latest-amd64

# ARM64 only (for Apple Silicon, ARM servers)
ghcr.io/nexyhub/n8n:nexyhub-latest-arm64
```

## Using Your NexyHub Docker Image

### Quick Start

```bash
# Pull the latest NexyHub-branded n8n
docker pull ghcr.io/nexyhub/n8n:nexyhub-latest

# Run it
docker run -it --rm \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  -e N8N_ENCRYPTION_KEY="your-encryption-key-here" \
  ghcr.io/nexyhub/n8n:nexyhub-latest
```

### Docker Compose

Create a `docker-compose.yml`:

```yaml
version: '3.8'

services:
  n8n:
    image: ghcr.io/nexyhub/n8n:nexyhub-latest
    container_name: n8n-nexyhub
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - N8N_PORT=5678
      - WEBHOOK_URL=${WEBHOOK_URL:-http://localhost:5678/}
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Then run:
```bash
docker-compose up -d
```

### Authentication for Private Images

If your repository is private, authenticate first:

```bash
# Create a Personal Access Token with read:packages scope
# Then login:
echo YOUR_GITHUB_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

## Workflow Details

### Build Process

1. **Checkout**: Gets the `nexyhub-branding` branch code
2. **Setup**: Installs Node.js, pnpm, and dependencies
3. **Build**: Runs `pnpm build:n8n` which compiles NexyHub theme
4. **Docker Build**: Creates platform-specific images
5. **Push**: Uploads to GitHub Container Registry
6. **Manifest**: Combines AMD64 and ARM64 into multi-arch images

### Build Time

- **AMD64 build**: ~8-12 minutes
- **ARM64 build**: ~8-12 minutes
- **Total**: ~15-20 minutes (builds run in parallel)

### Build Artifacts

Each successful build includes:
- Multi-arch Docker images
- Build summary in Actions tab
- Pull commands in job output
- Link to GitHub Packages

## Viewing Published Images

1. Go to your GitHub repository
2. Click on **Packages** (right sidebar)
3. Click on **n8n** package
4. You'll see all published tags and pull commands

Or visit directly:
`https://github.com/NexyHub/n8n/packages`

## Customization

### Build Environment Variables

Edit `.github/workflows/nexyhub-docker-build.yml` to customize:

```yaml
env:
  NODE_OPTIONS: '--max-old-space-size=7168'  # Memory for build
  NODE_VERSION: '22.21.0'                     # Node.js version
```

### Build Args

The workflow passes these to Docker:
- `N8N_VERSION=nexyhub-<sha>` - Version identifier
- `N8N_RELEASE_TYPE=nexyhub` - Release type
- `NODE_VERSION` - Node.js version

### Triggering on Other Branches

To build from other branches, edit the workflow trigger:

```yaml
on:
  push:
    branches:
      - nexyhub-branding
      - your-other-branch    # Add here
```

## Troubleshooting

### Build Fails with "No space left on device"

Increase the memory allocation in workflow:
```yaml
env:
  NODE_OPTIONS: '--max-old-space-size=8192'  # Increase from 7168
```

### Build Fails on ARM64

GitHub doesn't provide native ARM64 runners in the free tier. The workflow uses QEMU emulation which is slower but works. If builds timeout:
1. Increase timeout in workflow
2. Or remove ARM64 from matrix (AMD64 only)

### Can't Pull Image

Check authentication:
```bash
# Verify you can access the package
curl -H "Authorization: Bearer YOUR_TOKEN" \
  https://ghcr.io/v2/nexyhub/n8n/tags/list

# Re-authenticate
echo YOUR_TOKEN | docker login ghcr.io -u YOUR_USERNAME --password-stdin
```

### Theme Not Applied

Verify the build compiled the theme:
1. Check Actions logs for "Build n8n with NexyHub theme" step
2. Ensure no build errors
3. Verify `_tokens.scss` includes NexyHub imports

## Monitoring Builds

### GitHub Actions

1. Go to **Actions** tab
2. Click on workflow run
3. View detailed logs for each step
4. Check job summary for pull commands

### Notifications

GitHub sends notifications for:
- ✅ Successful builds
- ❌ Failed builds
- ⚠️ Workflow errors

Configure in: **Settings** → **Notifications**

## Best Practices

### 1. Test Before Pushing

```bash
# Build locally first
pnpm build:n8n

# Test the build
pnpm start
```

### 2. Use Specific Tags in Production

```bash
# ✅ Good - pinned to specific commit
docker pull ghcr.io/nexyhub/n8n:nexyhub-branding-a1b2c3d

# ⚠️ Caution - may change unexpectedly
docker pull ghcr.io/nexyhub/n8n:nexyhub-latest
```

### 3. Monitor Image Size

Check package size in GitHub Packages. If too large:
- Review what's included in `compiled/` directory
- Check for unnecessary node_modules
- Consider multi-stage Docker builds

### 4. Keep Workflow Updated

Periodically update GitHub Actions versions:
```bash
# Check for updates
gh api /repos/actions/checkout/releases/latest

# Update in workflow file
uses: actions/checkout@v5  # Update version
```

## CI/CD Integration

### GitHub Actions Integration

Create a deployment workflow that uses the NexyHub image:

```yaml
name: Deploy NexyHub n8n

on:
  workflow_run:
    workflows: ["NexyHub: Build and Push Docker Image"]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          # Your deployment script
          ssh user@server 'docker-compose pull && docker-compose up -d'
```

## Security Considerations

### 1. Image Scanning

The workflow uses Docker's built-in security scanning. View results in:
- GitHub Security tab
- Package security advisories

### 2. Secrets Management

Never commit secrets. Use GitHub Secrets:
1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Add secrets like `N8N_ENCRYPTION_KEY`
3. Reference in workflow: `${{ secrets.SECRET_NAME }}`

### 3. Package Permissions

Configure package access:
1. Go to **Package settings**
2. Set visibility (public/private)
3. Manage team access
4. Configure inheritance from repository

## Support

For issues with:
- **Workflow**: Check GitHub Actions logs
- **Theme**: See `docs/NEXYHUB_IMPLEMENTATION_SUMMARY.md`
- **Docker**: Check n8n's official Docker documentation
- **n8n**: Visit n8n community forum

## Related Documentation

- [NexyHub Git Workflow](NEXYHUB_GIT_WORKFLOW.md)
- [NexyHub Color Analysis](NEXYHUB_COLOR_ANALYSIS.md)
- [NexyHub Implementation Summary](NEXYHUB_IMPLEMENTATION_SUMMARY.md)
- [Theme README](../packages/frontend/@n8n/design-system/src/css/themes/nexyhub/README.md)
