# Phase 9: CI/CD Setup

## Overview

This phase sets up GitHub Actions workflows for automated building, testing, and deployment to AWS ECR with multi-architecture support.

## Prerequisites

- [ ] Phase 1-8 completed
- [ ] Docker build working
- [ ] Repository on GitHub

## Step-by-Step Instructions

### 1. Create GitHub Workflows Directory

```bash
mkdir -p .github/workflows
```

### 2. Create Build Workflow

Create `.github/workflows/build.yaml`:

```yaml
name: Build
on:
  pull_request:
    branches:
      - 'main'
  push:
    branches:
      - 'main'

env:
  REGISTRY_URL: '745662293263.dkr.ecr.eu-west-1.amazonaws.com/ribbon/app-haven-ribbon-{name}'

jobs:
  docker-build:
    name: Build docker image
    if: github.ref != 'refs/heads/main'
    runs-on: platform-runners-ecr
    concurrency:
      group: docker-build-${{ github.ref }}
      cancel-in-progress: true
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker buildx build . --progress=plain --secret id=npmrc,src=.npmrc_ci --build-arg GITHUB_PKG_TOKEN=${{ secrets.HAVEN_PLATFORM_BOT_PACKAGE_READ_PAT }} --build-arg APP_VERSION=0.0.0 -t github-actions-build

  generate-app-version:
    name: Generate APP_VERSION
    if: github.ref == 'refs/heads/main'
    runs-on: platform-runners-ecr
    outputs:
      APP_VERSION: ${{ steps.set_app_version.outputs.APP_VERSION }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set APP_VERSION
        id: set_app_version
        run: |
          BUILD_TIMESTAMP=$(date +'%y%m%d.%H%M');
          APP_VERSION="${BUILD_TIMESTAMP}-$(git rev-parse --short HEAD)"
          echo "APP_VERSION=${APP_VERSION}" >> $GITHUB_OUTPUT

  docker-publish:
    name: Build and publish docker image
    if: github.ref == 'refs/heads/main'
    needs: generate-app-version
    strategy:
      matrix:
        include: ${{ fromJSON(vars.PLATFORM_ECR_MULTI_ARCH_BUILD_MATRIX) }}
    runs-on: ${{ matrix.runner }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Docker registry
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker buildx build . \
          --platform ${{ matrix.platform }} \
          --progress=plain \
          --secret id=npmrc,src=.npmrc_ci \
          --build-arg GITHUB_PKG_TOKEN=${{ secrets.HAVEN_PLATFORM_BOT_PACKAGE_READ_PAT }} \
          --build-arg APP_VERSION=${{ needs.generate-app-version.outputs.APP_VERSION }} \
          -t ${{ env.REGISTRY_URL }}:${{ needs.generate-app-version.outputs.APP_VERSION }}-${{ matrix.arch }}

      - name: Push Docker image
        run: |
          docker push ${{ env.REGISTRY_URL }}:${{ needs.generate-app-version.outputs.APP_VERSION }}-${{ matrix.arch }}

      - uses: havenengineering/github-actions/CICD/store-multi-arch-image-artifact@v2.8.0
        with:
          UNIQUE_KEY: ${{ matrix.arch }}
          ARCH_IMAGE_FULL: ${{ env.REGISTRY_URL }}:${{ needs.generate-app-version.outputs.APP_VERSION }}-${{ matrix.arch }}

  publish-manifest:
    needs:
      - docker-publish
      - generate-app-version
    if: github.ref == 'refs/heads/main'
    runs-on: platform-runners-ecr
    concurrency:
      group: publish-manifest-${{ github.ref }}
      cancel-in-progress: true
    steps:
      - name: Set VERSION_TAG as env
        run: echo "VERSION_TAG=${{ needs.generate-app-version.outputs.APP_VERSION }}" >> $GITHUB_ENV

      - uses: havenengineering/github-actions/CICD/publish-multi-arch-manifest@v2.8.0
        with:
          REGISTRY_URL: ${{ env.REGISTRY_URL }}
          VERSION_TAG: ${{ env.VERSION_TAG }}

      - name: Create draft release
        uses: softprops/action-gh-release@v1
        with:
          draft: true
          name: ${{ needs.generate-app-version.outputs.APP_VERSION }}
          tag_name: ${{ needs.generate-app-version.outputs.APP_VERSION }}
          generate_release_notes: true
```

**Replace `{name}` with your actual app name** (e.g., `comms`, `venues`, etc.)

**Workflow behavior:**
- **On PR:** Builds Docker image to verify (doesn't push)
- **On main:** Builds multi-arch images, pushes to ECR, creates draft release

### 3. Configure GitHub Secrets

Your repository needs these secrets (ask DevOps/Platform team):

**Required Secrets:**
- `HAVEN_PLATFORM_BOT_PACKAGE_READ_PAT` - GitHub token for reading packages

**Required Variables:**
- `PLATFORM_ECR_MULTI_ARCH_BUILD_MATRIX` - Multi-arch build configuration

These should already be configured at the organization level.

### 4. Configure Branch Protection

Set up branch protection for `main`:

1. Go to GitHub repository settings
2. Navigate to Branches → Branch protection rules
3. Add rule for `main` branch:
   - Require pull request reviews before merging
   - Require status checks to pass before merging
   - Require branches to be up to date before merging
   - Required status checks: `docker-build`

### 5. Test the Workflow

**Test on PR:**
```bash
# Create a feature branch
git checkout -b feat/test-ci

# Make a small change
echo "# CI Test" >> README.md

# Commit and push
git add .
git commit -m "test: verify CI pipeline"
git push -u origin feat/test-ci

# Create PR on GitHub
# Watch Actions tab for build
```

**Verify:**
- `docker-build` job runs and succeeds
- No errors in build output
- All checks pass

**Test on main:**
```bash
# Merge the PR
# Watch Actions tab

# Verify:
# - generate-app-version job runs
# - docker-publish jobs run for each architecture
# - publish-manifest job runs
# - Draft release created
```

### 6. Create Release Process Documentation

Update `DEPLOYMENT.md` with release process:

```markdown
# Release Process

## Automatic Deployment to Dev

When a PR is merged to `main`:
1. CI builds multi-arch Docker images
2. Images pushed to ECR with version tag
3. Draft GitHub release created
4. Images automatically deployed to dev environment

## Production Release

To deploy to production:
1. Go to GitHub Releases
2. Find the draft release for your version
3. Edit the draft release:
   - Review release notes
   - Add any additional context
   - Mark as latest release
4. Publish the release
5. Production deployment triggers automatically

## Version Format

Versions follow the format: `YYMMDD.HHMM-{git-sha}`
Example: `241211.1430-a1b2c3d`

## Rollback

To rollback to a previous version:
1. Find the previous release tag
2. Re-deploy that image tag via AWS Console or CLI
3. Monitor application health
```

### 7. Add Status Badge to README

Update `README.md` to show build status:

```markdown
# App Haven Ribbon {Name}

[![Build](https://github.com/HavenEngineering/app-haven-ribbon-{name}/actions/workflows/build.yaml/badge.svg)](https://github.com/HavenEngineering/app-haven-ribbon-{name}/actions/workflows/build.yaml)

Haven Experience Admin application - {description}

## Development

\`\`\`bash
npm install
npm run dev
\`\`\`

## Deployment

See [DEPLOYMENT.md](./DEPLOYMENT.md) for deployment instructions.
```

### 8. Create Pull Request Template (Optional)

Create `.github/pull_request_template.md`:

```markdown
## Description
<!-- Describe your changes -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
<!-- Describe testing performed -->
- [ ] Tested locally
- [ ] Tested in Docker
- [ ] All tests pass

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No new warnings
- [ ] Tests added/updated

## Related Issues
<!-- Link related issues -->
Closes #
```

### 9. Commit CI/CD Configuration

```bash
git add .
git commit -m "feat: add GitHub Actions CI/CD pipeline

- Create build workflow for PRs and main branch
- Configure multi-arch Docker builds
- Set up ECR publishing
- Add automatic release creation
- Document release process
- Add build status badge

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

### 10. Verify End-to-End

Create a test PR to verify the entire pipeline:

1. Make a small change
2. Create PR
3. Verify build succeeds
4. Merge PR
5. Verify main build succeeds
6. Check ECR for new image
7. Verify draft release created

## Verification Checklist

- [ ] `.github/workflows/build.yaml` created
- [ ] Workflow has correct registry URL
- [ ] GitHub secrets verified/configured
- [ ] Branch protection configured
- [ ] PR build workflow tested
- [ ] Main build workflow tested
- [ ] Docker images appear in ECR
- [ ] Draft release created automatically
- [ ] Multi-arch builds succeed
- [ ] Documentation updated
- [ ] Build badge added to README

## Common Issues

**Issue: Workflow fails with "no basic auth credentials"**
- Verify `HAVEN_PLATFORM_BOT_PACKAGE_READ_PAT` secret exists
- Check secret has correct permissions
- Ensure secret is accessible to workflow

**Issue: "platform-runners-ecr" runner not available**
- This is a custom GitHub runner
- Contact DevOps if not available
- May need to use different runner type

**Issue: ECR push fails with permission denied**
- Verify runner has AWS credentials
- Check ECR repository exists
- Verify IAM permissions

**Issue: Multi-arch build matrix not found**
- Verify `PLATFORM_ECR_MULTI_ARCH_BUILD_MATRIX` variable exists
- Check organization-level variables
- Contact platform team for configuration

**Issue: Build succeeds but draft release not created**
- Check workflow permissions for releases
- Verify GITHUB_TOKEN has write permissions
- Check `softprops/action-gh-release` action configuration

## Next Steps

**[Phase 10: Testing & Finalization](./10-testing-finalization.md)**

## Notes for Claude

1. Replace `{name}` placeholder throughout
2. Verify registry URL with team
3. Test workflow on actual PR before finalizing
4. Ensure all secrets are configured
5. Don't commit secrets or tokens
6. Document any organization-specific changes
