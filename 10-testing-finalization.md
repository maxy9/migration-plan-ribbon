# Phase 10: Testing & Finalization

## Overview

This final phase ensures the migration is complete, tested, documented, and ready for production use.

## Prerequisites

- [ ] Phase 1-9 completed
- [ ] Application deployed to dev environment
- [ ] CI/CD pipeline working

## Step-by-Step Instructions

### 1. Comprehensive Functional Testing

Create a testing checklist based on source module features:

**Create `TESTING_CHECKLIST.md`:**
```markdown
# Testing Checklist

## Authentication
- [ ] Login flow works
- [ ] Token refresh works
- [ ] Logout works
- [ ] Session persists across page reloads

## Navigation
- [ ] All routes accessible
- [ ] Links navigate correctly
- [ ] Browser back/forward buttons work
- [ ] Deep links work
- [ ] 404 page shows for invalid routes

## Data Fetching
- [ ] Initial data loads on page mount
- [ ] Loading states display correctly
- [ ] Error states display correctly
- [ ] Empty states display correctly
- [ ] Data caches appropriately
- [ ] Stale data refetches

## Forms & Mutations
- [ ] Form validation works
- [ ] Form submission succeeds
- [ ] Success feedback shows
- [ ] Error handling works
- [ ] Optimistic updates work (if implemented)
- [ ] Data refreshes after mutation

## Park Context
- [ ] Park data loads from parent
- [ ] Park-specific queries work
- [ ] Park changes handled correctly

## PostMessage Communication
- [ ] paramChange events sent to parent
- [ ] pathChange events sent to parent
- [ ] navigate events received from parent
- [ ] setPark events received

## UI Components
- [ ] All buttons work
- [ ] Modals open and close
- [ ] Dropdowns work
- [ ] Date pickers work
- [ ] File uploads work (if applicable)
- [ ] Tables sort and filter
- [ ] Pagination works (if applicable)

## Styling
- [ ] All pages styled correctly
- [ ] Responsive design works
- [ ] No layout issues
- [ ] Icons display correctly
- [ ] Images load correctly

## Error Handling
- [ ] Network errors show user-friendly messages
- [ ] 401 errors trigger re-auth
- [ ] 404 errors show not found page
- [ ] 500 errors show error page
- [ ] Validation errors display inline

## Performance
- [ ] Initial page load < 3s
- [ ] No console errors
- [ ] No console warnings (or acceptable warnings documented)
- [ ] Images optimized
- [ ] Bundle size reasonable

## Browser Testing
- [ ] Chrome
- [ ] Firefox
- [ ] Safari
- [ ] Edge

## Embedded Context Testing
- [ ] Works in iframe
- [ ] Communication with parent works
- [ ] Routing with embedded prefix works
- [ ] No CORS issues
```

Work through this checklist systematically.

### 2. Compare with Source Module

**Side-by-side comparison:**

```bash
# Run both applications
# Source module (in parent app): https://admin.haven-dev.com/exp/comms
# New ribbon app: http://localhost:3000
```

**Verify:**
- All features from source exist in new app
- Behavior is identical
- UI is equivalent (can be styled differently)
- All edge cases handled

**Document differences:**
Create `MIGRATION_DIFFERENCES.md`:
```markdown
# Migration Differences

## Intentional Changes
- Styling updated to use Tailwind CSS
- Routing structure simplified
- Added React Query for better caching

## Known Limitations
- Feature X temporarily disabled (if any)
- Waiting for API Y to be updated (if any)

## To Be Completed Later
- Additional testing scenarios
- Performance optimizations
```

### 3. Performance Testing

**Check bundle size:**
```bash
npm run build

# Check output:
# Route (app)                              Size
# ├ ○ /                                    XXX kB
# ├ ○ /essential                           XXX kB
# └ ○ /marketing                           XXX kB
```

**Lighthouse audit:**
```bash
# Build and start production server
npm run build
npm start

# Run Lighthouse in Chrome DevTools
# Or use CLI:
npx lighthouse http://localhost:3000 --view
```

**Target scores:**
- Performance: > 80
- Accessibility: > 90
- Best Practices: > 90
- SEO: > 80

### 4. Security Review

**Check for common issues:**
- [ ] No secrets in code
- [ ] No console.log of sensitive data
- [ ] API calls use HTTPS
- [ ] XSS protection in place
- [ ] SQL injection not possible (use parameterized queries)
- [ ] CSRF protection (handled by API)
- [ ] Authentication required for all routes
- [ ] Authorization checked server-side

**Run security audit:**
```bash
npm audit

# Fix vulnerabilities
npm audit fix
```

### 5. Documentation Review

**Ensure documentation is complete:**

**README.md should include:**
- [ ] Project description
- [ ] Setup instructions
- [ ] Development commands
- [ ] Environment variables
- [ ] Build and deploy info
- [ ] Link to DEPLOYMENT.md

**Update README.md:**
```markdown
# App Haven Ribbon {Name}

[![Build](https://github.com/HavenEngineering/app-haven-ribbon-{name}/actions/workflows/build.yaml/badge.svg)](https://github.com/HavenEngineering/app-haven-ribbon-{name}/actions/workflows/build.yaml)

Next.js application for Haven Experience Admin - {description}

## Overview

This application provides {main features} for Haven parks. It runs as a standalone Next.js application and communicates with the parent application via postMessage API.

## Features

- Feature 1
- Feature 2
- Feature 3

## Tech Stack

- Next.js 15 (App Router)
- React 19
- TypeScript
- Tailwind CSS
- React Query (TanStack Query)
- MSAL for authentication
- Axios for API calls

## Prerequisites

- Node.js 24
- npm or yarn
- GITHUB_PKG_TOKEN for @havenengineering packages

## Setup

\`\`\`bash
# Clone repository
git clone https://github.com/HavenEngineering/app-haven-ribbon-{name}.git
cd app-haven-ribbon-{name}

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env.local
# Edit .env.local with your values

# Start development server
npm run dev
\`\`\`

Visit http://localhost:3000

## Development

\`\`\`bash
# Start dev server
npm run dev

# Run linter
npm run lint

# Build for production
npm run build

# Start production server
npm start

# Build Docker image
npm run docker:build

# Run Docker container
npm run docker:run
\`\`\`

## Environment Variables

See `.env.example` for required environment variables.

Key variables:
- `NEXT_PUBLIC_AZURE_AD_CLIENT_ID` - Azure AD client ID
- `NEXT_PUBLIC_AZURE_AD_TENANT_ID` - Azure AD tenant ID
- `NEXT_PUBLIC_API_BASE_URL` - API base URL

## Deployment

See [DEPLOYMENT.md](./DEPLOYMENT.md) for deployment instructions.

## Project Structure

\`\`\`
src/
├── app/              # Next.js App Router pages
├── components/       # React components
├── hooks/            # Custom React hooks
├── lib/              # Utility libraries
│   └── api/         # API client and services
├── msal/             # MSAL authentication config
├── providers/        # Context providers
└── utils/            # Utility functions
\`\`\`

## Testing

See [TESTING_CHECKLIST.md](./TESTING_CHECKLIST.md) for full testing checklist.

\`\`\`bash
# Run tests (when implemented)
npm test
\`\`\`

## Contributing

1. Create a feature branch
2. Make your changes
3. Test thoroughly
4. Create a Pull Request
5. Wait for review and CI checks
6. Merge after approval

## Migration Notes

This application was migrated from `app-haven-experience-admin-{name}` (React micro-frontend) to a standalone Next.js application.

See [MIGRATION_NOTES.md](./MIGRATION_NOTES.md) for migration details.

## Support

For issues or questions:
- Create a GitHub issue
- Contact the platform team

## License

Proprietary - Haven Engineering
```

### 6. Clean Up Code

**Remove development artifacts:**
- [ ] Remove test components (like NavTest)
- [ ] Remove debug console.logs
- [ ] Remove commented-out code
- [ ] Remove unused imports
- [ ] Remove unused files

**Run linter:**
```bash
npm run lint
npm run lint -- --fix
```

**Format code:**
```bash
npx prettier --write src/
```

### 7. Update Dependencies

**Check for updates:**
```bash
npm outdated

# Update carefully
npm update

# Test after updates
npm run build
npm run dev
```

**Verify:**
- Application still works
- No new warnings
- No breaking changes

### 8. Create Migration Summary

Create `MIGRATION_SUMMARY.md`:
```markdown
# Migration Summary

## Overview
Successfully migrated `app-haven-experience-admin-{name}` to `app-haven-ribbon-{name}`.

## Migration Date
December 2024

## Team
- Developers: [Names]
- Reviewers: [Names]

## What Changed

### Architecture
- **Before:** React micro-frontend module built with Vite
- **After:** Standalone Next.js 15 application with App Router

### Key Technical Changes
- React Router v5 → Next.js App Router
- SCSS Modules → Tailwind CSS
- Direct API calls → React Query
- npm package → Docker container
- Hosted in parent app → Standalone with iframe

### Deployment
- **Before:** Published to GitHub Package Registry
- **After:** Deployed to AWS ECR as Docker image

## Features Migrated
- [x] Feature 1
- [x] Feature 2
- [x] Feature 3

## Testing Completed
- [x] Functional testing
- [x] Integration testing
- [x] Browser compatibility testing
- [x] Performance testing
- [x] Security review

## Known Issues
None

## Production Readiness
✅ Ready for production

## Rollback Plan
If issues occur, can rollback to original module at any time.

## Next Steps
1. Deploy to production
2. Monitor for issues
3. Gather user feedback
4. Plan future enhancements
```

### 9. Final Code Review

**Self-review checklist:**
- [ ] All TODO comments addressed or removed
- [ ] No hardcoded values (use env vars)
- [ ] No sensitive data in code
- [ ] All functions have clear purposes
- [ ] Complex logic has comments
- [ ] TypeScript types are accurate
- [ ] Error handling is comprehensive
- [ ] Loading states are user-friendly
- [ ] Accessibility considered (ARIA labels, keyboard nav)

**Request team review:**
- Create PR for final review
- Address feedback
- Get approval from at least 2 reviewers

### 10. Production Deployment Preparation

**Pre-production checklist:**
- [ ] All tests passing
- [ ] CI/CD pipeline green
- [ ] Documentation complete
- [ ] Code reviewed and approved
- [ ] Performance acceptable
- [ ] Security review passed
- [ ] Stakeholders notified
- [ ] Rollback plan documented

**Deploy to production:**
1. Merge final PR to main
2. Wait for CI to build and push to ECR
3. Find draft release in GitHub
4. Review release notes
5. Publish release
6. Monitor deployment
7. Verify in production
8. Monitor for issues

### 11. Post-Deployment Monitoring

**Monitor for first 24 hours:**
- Application logs (CloudWatch)
- Error rates
- API response times
- User feedback
- Authentication issues

**Create incident response plan:**
```markdown
# Incident Response

If issues occur:

1. **Identify severity:**
   - P1 (Critical): Application down
   - P2 (High): Major feature broken
   - P3 (Medium): Minor feature issue
   - P4 (Low): Cosmetic issue

2. **Response:**
   - P1: Immediate rollback
   - P2: Fix within 2 hours or rollback
   - P3: Fix in next deployment
   - P4: Fix when convenient

3. **Rollback procedure:**
   - Redeploy previous version tag
   - Monitor for stability
   - Investigate root cause

4. **Communication:**
   - Notify users of issues
   - Update status page
   - Post-mortem after resolution
```

### 12. Final Commit

```bash
git add .
git commit -m "chore: finalize migration

- Complete all testing
- Update documentation
- Clean up code
- Prepare for production

Migration of app-haven-experience-admin-{name} to app-haven-ribbon-{name} complete.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

## Verification Checklist

Final verification before marking migration complete:

- [ ] All 10 phases completed
- [ ] Application fully functional
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Code clean and reviewed
- [ ] CI/CD pipeline working
- [ ] Security review passed
- [ ] Performance acceptable
- [ ] Deployed to dev successfully
- [ ] Ready for production

## Success Criteria

Migration is successful when:

✅ **Functional parity:** All features from original module work in new app
✅ **Quality:** No critical bugs, acceptable performance
✅ **Documentation:** Complete and accurate
✅ **Deployment:** Automated and reliable
✅ **Team confidence:** Team comfortable supporting the new application

## Celebration

🎉 **Migration Complete!**

You have successfully migrated a React micro-frontend module to a modern Next.js standalone application.

**Achievements:**
- ✅ Modern architecture (Next.js 15 with App Router)
- ✅ Better performance (React Query caching)
- ✅ Independent deployment (Docker + ECR)
- ✅ Automated CI/CD (GitHub Actions)
- ✅ Improved developer experience

**Share your success:**
- Update the team
- Document lessons learned
- Share knowledge for next migration

## Next Migration

Use this plan as a template for migrating the next admin module!

**Improvements for next time:**
- Note what worked well
- Note what could be improved
- Update this plan with learnings

## Notes for Claude

When executing this phase:
1. Be thorough with testing - this is the final gate
2. Don't skip documentation - future developers will thank you
3. Test in production-like environment
4. Get team review before marking complete
5. Monitor closely after deployment
6. Document everything learned
7. Celebrate the achievement!

---

**Thank you for using this migration plan!**

If you have feedback or suggestions for improving this plan, please contribute back to help future migrations.
