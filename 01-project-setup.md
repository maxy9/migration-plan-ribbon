# Phase 1: Project Setup & Initialization

## Overview

This phase creates the foundational structure for the new NextJS application. You'll create a new repository, initialize the NextJS project, and set up the basic directory structure.

## Prerequisites

Before starting:
- [ ] Identify the source module to migrate (e.g., `app-haven-experience-admin-{name}`)
- [ ] Determine the new app name (e.g., `app-haven-ribbon-{name}`)
- [ ] Have access to GitHub and necessary permissions

## Step-by-Step Instructions

### 1. Explore the Source Module

First, understand what you're migrating:

```bash
# Navigate to and explore the source module
cd app-haven-experience-admin-{name}
```

**Tasks:**
- Read the source module's `README.md`
- Explore `src/` directory structure
- Note the main sections/features (look in `src/sections/` or main components)
- Review `package.json` to understand dependencies
- Identify key business logic areas

**Document your findings:**
- What are the main features/sections?
- What external APIs does it call?
- What shared packages does it use?
- What business-critical logic exists?

### 2. Create Project Directory

Create a new directory for the ribbon app:

```bash
# Create directory in your Git workspace
mkdir app-haven-ribbon-{name}
cd app-haven-ribbon-{name}
```

**Note:** Git repository creation will be handled in Phase 10 after the migration is complete.

### 3. Initialize NextJS Project

Create a new Next.js project with the App Router:

```bash
# If you're in an empty cloned repository:
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir --import-alias "@/*"

# Answer the prompts:
# ✔ Would you like to use TypeScript? Yes
# ✔ Would you like to use ESLint? Yes
# ✔ Would you like to use Tailwind CSS? Yes
# ✔ Would you like your code inside a `src/` directory? Yes
# ✔ Would you like to use App Router? Yes
# ✔ Would you like to use Turbopack for next dev? No
# ✔ Would you like to customize the import alias? Yes (@/*)
```

### 4. Update package.json

Edit `package.json` to match the project structure:

```json
{
  "name": "app-haven-ribbon-{name}",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "dev-cert": "next dev --experimental-https",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

**Key points:**
- Name should be `app-haven-ribbon-{name}` (not a scoped package)
- Mark as `"private": true` (not published to npm)
- Keep the standard Next.js scripts
- Add `dev-cert` script for local HTTPS testing if needed

### 5. Configure Node Version

Create `.nvmrc` file to match the Docker base image:

```bash
echo "24" > .nvmrc
```

This ensures consistency between local development and Docker builds.

### 6. Set Up Git Configuration

Create `.gitignore`:

```
# dependencies
/node_modules
/.pnp
.pnp.js

# testing
/coverage

# next.js
/.next/
/out/

# production
/build

# misc
.DS_Store
*.pem

# debug
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# local env files
.env*.local
.env

# vercel
.vercel

# typescript
*.tsbuildinfo
next-env.d.ts

# IDE
.idea
.vscode
*.swp
*.swo
```

### 7. Configure npm Registry Access

Create `.npmrc` for local development:

```
@havenengineering:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_PKG_TOKEN}
```

Create `.npmrc_ci` for CI builds:

```
@havenengineering:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_PKG_TOKEN}
always-auth=true
```

**Important:** Make sure your local environment has `GITHUB_PKG_TOKEN` set with read access to Haven packages.

### 8. Create Basic Directory Structure

Set up the standard directory structure:

```bash
# Create directories that will be needed
mkdir -p src/components
mkdir -p src/hooks
mkdir -p src/lib
mkdir -p src/msal
mkdir -p src/providers
mkdir -p src/utils
mkdir -p public
```

### 9. Save Your Progress

At this point, the basic project structure is set up. Git initialization and commits will be handled in Phase 10 after the entire migration is complete.

## Verification Checklist

Before moving to Phase 2, verify:

- [ ] Project directory created
- [ ] Next.js project initialized with TypeScript
- [ ] `package.json` has correct name and is marked private
- [ ] `.nvmrc` is set to version 24
- [ ] `.npmrc` and `.npmrc_ci` configured for GitHub packages
- [ ] Basic directory structure created
- [ ] Project runs locally with `npm run dev`

## Test the Setup

Run these commands to verify everything works:

```bash
# Install dependencies
npm install

# Start dev server
npm run dev

# In another terminal, verify build works
npm run build
```

Visit http://localhost:3000 - you should see the default Next.js welcome page.

## Common Issues

**Issue: npm install fails with 401 Unauthorized**
- Ensure `GITHUB_PKG_TOKEN` environment variable is set
- Verify token has `read:packages` permission
- Check `.npmrc` file exists and is correctly formatted

**Issue: Port 3000 already in use**
- Run `PORT=3001 npm run dev` to use a different port
- Or stop the conflicting process

**Issue: TypeScript errors on first build**
- This is normal, we'll configure TypeScript in Phase 2

## Next Steps

Once Phase 1 is complete and verified, proceed to:
**[Phase 2: Configuration Files](./02-configuration-files.md)**

## Notes for Claude

When executing this phase:
1. Replace `{name}` with the actual module name being migrated
2. Explore the source module first to understand what's being migrated
3. Document any unique aspects of the source module
4. Verify each step completes successfully before moving to the next
5. If you encounter module-specific needs, document them for later phases
