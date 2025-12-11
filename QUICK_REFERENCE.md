# Quick Reference Guide

This is a condensed reference for the Haven Admin to Ribbon migration process. For detailed instructions, see the individual phase documents.

## Migration Phases

| Phase | Focus | Key Deliverables | Time Estimate |
|-------|-------|------------------|---------------|
| [1. Project Setup](./01-project-setup.md) | Initialize NextJS project | Repository, basic structure | 1-2 hours |
| [2. Configuration](./02-configuration-files.md) | Build tools & config | next.config, tsconfig, env files | 2-3 hours |
| [3. Authentication](./03-authentication-setup.md) | MSAL integration | Auth providers, login flow | 2-3 hours |
| [4. Providers](./04-provider-architecture.md) | Context architecture | Query, Park, Auth providers | 2-3 hours |
| [5. Routing](./05-routing-migration.md) | App Router structure | Pages, layouts, navigation | 4-6 hours |
| [6. Components](./06-component-migration.md) | Business logic | Migrated components | 8-16 hours |
| [7. API & Data](./07-api-data-fetching.md) | React Query setup | API client, query hooks | 4-6 hours |
| [8. Docker](./08-docker-deployment.md) | Containerization | Dockerfile, build process | 2-3 hours |
| [9. CI/CD](./09-cicd-setup.md) | GitHub Actions | Automated deployment | 2-3 hours |
| [10. Testing](./10-testing-finalization.md) | QA & launch | Production ready | 4-8 hours |

**Total Estimated Time: 30-50 hours** (varies by module complexity)

## Key Commands

### Development
```bash
npm install              # Install dependencies
npm run dev             # Start dev server
npm run build           # Build for production
npm start               # Start production server
npm run lint            # Run linter
```

### Docker
```bash
npm run docker:build    # Build Docker image
npm run docker:run      # Run Docker container
docker-compose up       # Run with compose
```

### Git
```bash
git checkout -b feat/my-feature  # Create feature branch
git commit -m "feat: description"  # Commit with message
git push                          # Push to remote
```

## Key Conversions

### Routing
| React Router v5 | Next.js App Router |
|-----------------|-------------------|
| `useHistory()` | `useRouter()` from `next/navigation` |
| `useLocation()` | `usePathname()` + `useSearchParams()` |
| `useParams()` | `params` prop (await in server components) |
| `history.push('/path')` | `router.push('/path')` |
| `<Link to="/path">` | `<Link href="/path">` |
| `<Redirect to="/path">` | `redirect('/path')` or `router.push()` |

### Component Patterns
```typescript
// Always add for client-side features
'use client';

// Dynamic params in App Router
export default function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = use(params);  // Next.js 15
}

// Search params
const searchParams = useSearchParams();
const value = searchParams.get('key');

// Or with nuqs
const [status, setStatus] = useQueryState('status');
```

### Data Fetching
```typescript
// Query (GET)
const { data, isLoading, error } = useQuery({
  queryKey: ['resource', id],
  queryFn: () => fetchResource(id),
});

// Mutation (POST/PUT/DELETE)
const mutation = useMutation({
  mutationFn: createResource,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['resources'] });
  },
});
```

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| "use is not a function" | Import `use` from React, params are Promises in Next.js 15 |
| Hydration mismatch | Add `'use client'`, check server/client rendering |
| SVG imports fail | Verify `global.d.ts` exists, @svgr/webpack installed |
| Auth popup blocked | Use redirect flow or instruct user to allow popups |
| API 401 errors | Check MSAL token acquisition and Bearer token format |
| Tailwind not working | Verify globals.css has directives, restart dev server |
| Docker build fails | Check .npmrc_ci exists, GITHUB_PKG_TOKEN is passed |

## File Structure Template

```
app-haven-ribbon-{name}/
├── .github/
│   └── workflows/
│       └── build.yaml           # CI/CD pipeline
├── public/                      # Static assets
├── src/
│   ├── app/                     # Next.js pages
│   │   ├── layout.tsx          # Root layout
│   │   ├── page.tsx            # Home page
│   │   ├── globals.css         # Global styles
│   │   └── {section}/          # Section routes
│   │       ├── layout.tsx      # Section layout
│   │       └── page.tsx        # Section page
│   ├── components/             # React components
│   │   ├── shared/             # Reusable components
│   │   └── {section}/          # Section-specific
│   ├── hooks/                  # Custom hooks
│   ├── lib/                    # Libraries & utilities
│   │   └── api/                # API client
│   │       ├── client.ts       # Axios instance
│   │       └── {domain}.ts     # Domain services
│   ├── msal/                   # MSAL config
│   │   ├── msalConfig.ts       # Configuration
│   │   └── MsalInstance.ts     # Singleton instance
│   ├── providers/              # Context providers
│   │   ├── Providers.tsx       # Root provider
│   │   ├── AuthProvider.tsx    # Auth context
│   │   ├── ParkProvider.tsx    # Park context
│   │   └── QueryProvider.tsx   # React Query
│   └── utils/                  # Utility functions
├── .env.production             # Production env vars
├── .env.example                # Env template
├── .gitignore                  # Git ignore rules
├── .npmrc                      # npm config (local)
├── .npmrc_ci                   # npm config (CI)
├── Dockerfile                  # Docker build
├── next.config.ts              # Next.js config
├── package.json                # Dependencies
├── README.md                   # Project docs
├── tailwind.config.ts          # Tailwind config
└── tsconfig.json               # TypeScript config
```

## Environment Variables

```bash
# .env.production
NEXT_PUBLIC_AZURE_AD_CLIENT_ID=75de59c0-2a7f-4d4a-baa6-fcc8987e1615
NEXT_PUBLIC_AZURE_AD_TENANT_ID=6bec8931-53ed-4238-a6a8-495fa78471d5
NEXT_PUBLIC_API_BASE_URL=https://api.haven.com
```

## Dependencies Checklist

Essential dependencies for all Ribbon apps:
- [x] `next` (15+)
- [x] `react` (19)
- [x] `react-dom` (19)
- [x] `typescript`
- [x] `tailwindcss`
- [x] `@azure/msal-react` (authentication)
- [x] `@tanstack/react-query` (data fetching)
- [x] `nuqs` (URL state)
- [x] `axios` (HTTP client)
- [x] `dayjs` (dates)
- [x] `@svgr/webpack` (SVG handling)

## PostMessage API

Messages sent to parent:
```typescript
// Parameter changes
{ type: 'paramChange', value: 'param1=value1&param2=value2' }

// Path changes
{ type: 'pathChange', value: { appName: 'comms', url: 'essential/park/page' } }

// Request park data
{ type: 'getPark' }
```

Messages received from parent:
```typescript
// Navigate to path
{ type: 'navigate', value: '/essential' }

// Set park data
{ type: 'setPark', value: { id: 'park-123', name: 'Park Name' } }
```

## Git Commit Conventions

```bash
feat: new feature
fix: bug fix
docs: documentation changes
style: formatting, white-space
refactor: code restructuring
test: adding tests
chore: maintenance tasks
```

Always include footer:
```
🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
```

## Checklist Progress Tracker

Use this to track your migration:

- [ ] Phase 1: Project Setup
- [ ] Phase 2: Configuration Files
- [ ] Phase 3: Authentication Setup
- [ ] Phase 4: Provider Architecture
- [ ] Phase 5: Routing Migration
- [ ] Phase 6: Component Migration
- [ ] Phase 7: API & Data Fetching
- [ ] Phase 8: Docker & Deployment
- [ ] Phase 9: CI/CD Setup
- [ ] Phase 10: Testing & Finalization

---

**For detailed instructions, see [README.md](./README.md) and individual phase documents.**
