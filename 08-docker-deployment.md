# Phase 8: Docker & Deployment

## Overview

This phase creates the Docker configuration for containerizing and deploying the Next.js application to AWS ECR.

## Prerequisites

- [ ] Phase 1-7 completed
- [ ] Application functional locally
- [ ] All features tested

## Step-by-Step Instructions

### 1. Create Dockerfile

Create `Dockerfile` in project root:

```dockerfile
ARG BASE_IMAGE=public.ecr.aws/docker/library/node
ARG DUMB_INIT_VER=~1.2.5
ARG BASE_IMAGE_VERSION=24-alpine

# Installs dependencies
FROM ${BASE_IMAGE}:${BASE_IMAGE_VERSION} AS install
ARG GITHUB_PKG_TOKEN
WORKDIR /app
COPY --chown=node:node package*.json ./
RUN --mount=type=secret,mode=0644,id=npmrc,target=/app/.npmrc npm ci --force
COPY --chown=node:node . .
RUN npm run lint

# Built app
FROM ${BASE_IMAGE}:${BASE_IMAGE_VERSION} AS build
ARG BUILD_ID
ARG APP_VERSION
ENV NEXT_PUBLIC_BUILD_ID="${BUILD_ID}"
ENV NEXT_PUBLIC_DD_VERSION="${APP_VERSION}"
ENV APP_ENV=production
WORKDIR /app
COPY --chown=node:node --from=install /app .
RUN npm run build
RUN npm prune --omit=dev --force

# Stage 2
# This stage extracts the build from stage 1 to produce a lean final image
FROM ${BASE_IMAGE}:${BASE_IMAGE_VERSION} AS release
ARG DUMB_INIT_VER
ARG API_VERSION
ARG BUILD_ID
ARG APP_VERSION
ENV BUILD_ID=${BUILD_ID}
ENV NEXT_PUBLIC_DD_VERSION="${APP_VERSION}"
ENV NEXT_PUBLIC_BUILD_ID="${BUILD_ID}"
ENV API_VERSION=$API_VERSION
ENV APP_ENV=production
RUN apk --no-cache add dumb-init=$DUMB_INIT_VER
WORKDIR /home/site/wwwroot
COPY --chown=node:node --from=build /app/package.json ./package.json
COPY --chown=node:node --from=build /app/node_modules ./node_modules
COPY --chown=node:node --from=build /app/.next ./.next
COPY --chown=node:node --from=build /app/public ./public

COPY --chown=node:node --from=build /app/.env.production ./.env.production
COPY --chown=node:node --from=build /app/next.config.ts ./next.config.ts
EXPOSE 3000
USER node
CMD [ "dumb-init", "npm", "start" ]
```

**Dockerfile stages:**
1. **install** - Install dependencies and run linter
2. **build** - Build the Next.js application
3. **release** - Minimal production image

**Key features:**
- Multi-stage build for smaller final image
- Runs as non-root user (node)
- Uses dumb-init for proper signal handling
- Includes build metadata (BUILD_ID, APP_VERSION)

### 2. Create .dockerignore

Create `.dockerignore`:

```
node_modules
.next
.git
.github
*.md
.env
.env.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
.vscode
.idea
```

### 3. Test Docker Build Locally

```bash
# Build locally
docker build . \
  --secret id=npmrc,src=.npmrc \
  --build-arg GITHUB_PKG_TOKEN=$GITHUB_PKG_TOKEN \
  --build-arg APP_VERSION=local-test \
  -t app-haven-ribbon-{name}:local

# Run locally
docker run -p 3000:3000 app-haven-ribbon-{name}:local

# Visit http://localhost:3000
```

### 4. Add Build Scripts to package.json

Update `package.json`:

```json
{
  "scripts": {
    "dev": "next dev",
    "dev-cert": "next dev --experimental-https",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "docker:build": "docker build . --secret id=npmrc,src=.npmrc --build-arg GITHUB_PKG_TOKEN=$GITHUB_PKG_TOKEN --build-arg APP_VERSION=local -t app-haven-ribbon-{name}:local",
    "docker:run": "docker run -p 3000:3000 app-haven-ribbon-{name}:local"
  }
}
```

Replace `{name}` with your app name.

### 5. Create Docker Compose (Optional for Local Development)

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - GITHUB_PKG_TOKEN=${GITHUB_PKG_TOKEN}
        - APP_VERSION=local
      secrets:
        - npmrc
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    volumes:
      - ./.env.production:/home/site/wwwroot/.env.production

secrets:
  npmrc:
    file: ./.npmrc
```

Usage:
```bash
docker-compose up --build
```

### 6. Verify Production Build

Test the production build locally:

```bash
# Build for production
npm run build

# Start production server
npm start

# Visit http://localhost:3000
# Test all functionality
```

**Check for:**
- All pages load correctly
- Authentication works
- API calls succeed
- No console errors
- Assets load properly

### 7. Optimize Next.js Build

Update `next.config.ts` for production optimizations:

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  assetPrefix: '/embedded/{app-name}',
  async rewrites() {
    return [
      {
        source: '/embedded/{app-name}/:path*',
        destination: '/:path*',
      },
    ];
  },
  images: {
    remotePatterns: [new URL('https://images.ctfassets.net/**')],
  },
  // Production optimizations
  compress: true,
  poweredByHeader: false,
  reactStrictMode: true,

  webpack(config) {
    // SVG handling
    const fileLoaderRule = config.module.rules.find(
      (rule: { test: { test: (arg0: string) => never } }) => rule.test?.test?.('.svg')
    );

    config.module.rules.push(
      {
        ...fileLoaderRule,
        test: /\.svg$/i,
        resourceQuery: /url/,
      },
      {
        test: /\.svg$/i,
        issuer: fileLoaderRule.issuer,
        resourceQuery: { not: [...fileLoaderRule.resourceQuery.not, /url/] },
        use: ['@svgr/webpack'],
      }
    );

    fileLoaderRule.exclude = /\.svg$/i;

    return config;
  },
};

export default nextConfig;
```

### 8. Document Deployment Process

Create `DEPLOYMENT.md`:

```markdown
# Deployment Guide

## Prerequisites

- AWS credentials configured
- Access to ECR repository
- GitHub secrets configured

## ECR Repository

**Registry:** `745662293263.dkr.ecr.eu-west-1.amazonaws.com/ribbon/app-haven-ribbon-{name}`

## Build Process

Builds are triggered automatically:
- On PR: Build only (verification)
- On main: Build, tag, and push to ECR

## Manual Deployment

If needed to deploy manually:

\`\`\`bash
# Login to ECR
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 745662293263.dkr.ecr.eu-west-1.amazonaws.com

# Build
docker build . \\
  --secret id=npmrc,src=.npmrc_ci \\
  --build-arg GITHUB_PKG_TOKEN=$GITHUB_PKG_TOKEN \\
  --build-arg APP_VERSION=$(date +'%y%m%d.%H%M')-$(git rev-parse --short HEAD) \\
  -t 745662293263.dkr.ecr.eu-west-1.amazonaws.com/ribbon/app-haven-ribbon-{name}:latest

# Push
docker push 745662293263.dkr.ecr.eu-west-1.amazonaws.com/ribbon/app-haven-ribbon-{name}:latest
\`\`\`

## Monitoring

- Check GitHub Actions for build status
- Monitor application logs in CloudWatch
- Verify deployment in AWS Console
```

### 9. Docker Configuration Complete

Docker setup is complete. Continue to Phase 9 to set up CI/CD.

## Verification Checklist

- [ ] Dockerfile created
- [ ] .dockerignore created
- [ ] Docker builds successfully locally
- [ ] Docker container runs successfully
- [ ] Application accessible in container
- [ ] Production build works locally
- [ ] All features work in production mode
- [ ] Environment variables loaded correctly
- [ ] Assets served correctly
- [ ] No build warnings or errors

## Common Issues

**Issue: npm install fails in Docker**
- Ensure `.npmrc_ci` exists
- Verify GITHUB_PKG_TOKEN is passed as build arg
- Check secret mounting syntax

**Issue: Build fails with memory error**
- Increase Docker memory allocation
- Use `--max-old-space-size` flag if needed

**Issue: Container exits immediately**
- Check logs: `docker logs <container-id>`
- Verify CMD in Dockerfile is correct
- Ensure port 3000 is not already in use

**Issue: Environment variables not loaded**
- Verify `.env.production` is copied in Dockerfile
- Check Next.js loads env files correctly
- Ensure variables are prefixed with `NEXT_PUBLIC_`

## Next Steps

**[Phase 9: CI/CD Setup](./09-cicd-setup.md)**

## Notes for Claude

1. Test Docker build before committing
2. Verify container runs and app is accessible
3. Check production build has no errors
4. Replace `{name}` placeholders with actual app name
5. Ensure secrets are never committed
