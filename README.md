# Migration Plan: Admin Module to Ribbon NextJS App

This directory contains a comprehensive step-by-step plan for migrating Haven `app-haven-experience-admin-*` modules to `app-haven-ribbon-*` NextJS applications.

## Overview

This migration transforms React micro-frontend modules (built with Vite, designed to be hosted in a parent application) into standalone NextJS 15 applications that run in Docker containers and communicate with a parent application via postMessage API.

## Migration Reference

**Source Example:** `app-haven-experience-admin-comms`
- React micro-frontend module
- Built with Vite as library
- Published as npm package
- Hosted within parent React app
- Uses React Router v5

**Target Example:** `app-haven-ribbon-comms`
- Standalone NextJS 15 application
- Dockerized deployment to ECR
- Embedded at `/embedded/{app-name}` path
- Uses Next.js App Router
- Communicates via postMessage

## Important: Git Workflow

**Git initialization and commits are handled at the END of the migration (Phase 10), not after each phase.**

This allows you to focus on the migration work without interruption. Once everything is complete and tested, you'll initialize git, create a single comprehensive commit, and push to GitHub.

## Migration Phases

The migration is broken down into 10 phases that should be completed in order:

### Phase 1: Project Setup & Initialization
[`01-project-setup.md`](./01-project-setup.md)
- Create NextJS project structure
- Initialize repository
- Set up basic configuration

### Phase 2: Configuration Files
[`02-configuration-files.md`](./02-configuration-files.md)
- Next.js configuration
- TypeScript setup
- Build tools (Tailwind, PostCSS, ESLint)
- Environment files

### Phase 3: Authentication Setup
[`03-authentication-setup.md`](./03-authentication-setup.md)
- MSAL integration
- Azure AD configuration
- Authentication providers

### Phase 4: Provider Architecture
[`04-provider-architecture.md`](./04-provider-architecture.md)
- Root providers
- React Query setup
- Park and session providers
- PostMessage communication

### Phase 5: Routing Migration
[`05-routing-migration.md`](./05-routing-migration.md)
- Convert React Router to App Router
- Page and layout structure
- Navigation updates

### Phase 6: Component Migration
[`06-component-migration.md`](./06-component-migration.md)
- Component structure migration
- Styling conversion (SCSS to Tailwind)
- Asset handling

### Phase 7: API & Data Fetching
[`07-api-data-fetching.md`](./07-api-data-fetching.md)
- API client setup
- React Query integration
- Data fetching patterns

### Phase 8: Docker & Deployment
[`08-docker-deployment.md`](./08-docker-deployment.md)
- Dockerfile creation
- Multi-stage builds
- Container configuration

### Phase 9: CI/CD Setup
[`09-cicd-setup.md`](./09-cicd-setup.md)
- GitHub Actions workflows
- ECR deployment
- Multi-arch builds

### Phase 10: Testing & Finalization
[`10-testing-finalization.md`](./10-testing-finalization.md)
- Testing strategy
- Documentation updates
- Final cleanup

## How to Use This Plan

### For Claude (AI Assistant)

When starting a migration, follow these instructions:

1. **Read this README first** to understand the overall context
2. **Work through phases sequentially** - each phase builds on the previous
3. **Before starting each phase:**
   - Read the phase document completely
   - Understand the source module structure by exploring the `app-haven-experience-admin-{module}` repository
   - Identify specific business logic that needs to be preserved
4. **During each phase:**
   - Follow the detailed instructions
   - Adapt examples to the specific module being migrated
   - Preserve all business logic from the original module
   - Test changes incrementally
5. **After each phase:**
   - Verify the work is complete
   - Test that everything works
   - Document any deviations or module-specific changes

### For Human Developers

1. Review the entire plan to understand the scope
2. Clone or reference both the source admin module and the example ribbon-comms app
3. Follow phases in order, checking off tasks as you complete them
4. Refer to the source repositories as needed:
   - `app-haven-experience-admin-comms` (source example)
   - `app-haven-ribbon-comms` (target example)

## Key Differences to Remember

| Aspect | Admin Module | Ribbon App |
|--------|-------------|------------|
| **Framework** | React + Vite | Next.js 15 |
| **Routing** | React Router v5 | App Router |
| **Distribution** | npm package | Docker container |
| **Hosting** | Parent React app | Standalone with iframe |
| **Styling** | SCSS modules | Tailwind CSS |
| **Build Output** | ES/CJS modules | Optimized Next.js build |
| **Deployment** | npm registry | ECR registry |
| **Communication** | Direct imports | postMessage API |

## Important Notes

- **Preserve Business Logic**: The core functionality must remain identical
- **Test Thoroughly**: Each phase should be tested before moving to the next
- **Module-Specific Adaptations**: Some modules may require unique handling - document these
- **Dependencies**: Keep shared `@havenengineering` packages where appropriate
- **Authentication**: All apps use the same MSAL configuration pattern
- **Embedded Routing**: All ribbon apps are hosted at `/embedded/{app-name}`

## Questions or Issues?

If you encounter issues or have questions about the migration:
- Review the example repositories (admin-comms and ribbon-comms)
- Check if the issue is module-specific or a general pattern
- Document any deviations from the standard pattern
- Update these instructions if you discover better approaches
