# Phase 5: Routing Migration

## Overview

This phase migrates routing from React Router v5 to Next.js App Router. This is one of the most critical phases as it involves restructuring the application's navigation and page architecture.

## Prerequisites

- [ ] Phase 1-4 completed
- [ ] Understanding of source module's routing structure
- [ ] Identified all routes in the original module

## Step-by-Step Instructions

### 1. Analyze Source Routing Structure

**Explore the source module's routing:**

```bash
# In the source module directory
# Find routing files
find src -name "routes.ts" -o -name "routes.tsx" -o -name "*Route*"

# Look at the main entry point
cat src/index.tsx

# Examine section routing
cat src/sections/*/routes.ts
```

**Document the routing:**
- What are the main routes?
- Are there nested routes?
- What are the dynamic segments (`:param`)?
- What headers/layouts are associated with each route?

**Example from admin-comms:**
- `/comms/emergency` - Emergency section root
- `/comms/emergency/:parkCode/:innerRoute` - Park-specific emergency pages
- `/comms/marketing` - Marketing section root

### 2. Plan App Router Structure

Next.js App Router uses file system-based routing. Plan your directory structure:

**React Router Route** → **Next.js Path**
- `/comms/emergency` → `src/app/essential/page.tsx`
- `/comms/marketing` → `src/app/marketing/page.tsx`
- `/comms/emergency/:parkCode/:innerRoute` → `src/app/essential/[parkCode]/[innerRoute]/page.tsx`

**General conversion rules:**
- Routes become directories in `src/app/`
- Each route directory has a `page.tsx` (the page component)
- Layouts are defined in `layout.tsx`
- Dynamic segments use `[param]` folder names
- Catch-all routes use `[...param]`

### 3. Create Base Route Structure

Based on your analysis, create the directory structure. For the comms example:

```bash
mkdir -p src/app/essential
mkdir -p src/app/marketing
```

**Note:** Route names may differ from the original (e.g., "emergency" → "essential"). Use names that make sense for the embedded context.

### 4. Create Root Page (Landing/Redirect)

Update `src/app/page.tsx` as a landing page or redirect:

```typescript
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export default function Home() {
  const router = useRouter();

  useEffect(() => {
    // Redirect to default section
    router.push('/essential');
  }, [router]);

  return (
    <div className="flex items-center justify-center min-h-screen">
      <div>Loading...</div>
    </div>
  );
}
```

Or create a proper landing page if needed.

### 5. Create Section Pages

For each main section, create a `page.tsx`.

**Example: `src/app/essential/page.tsx`**

```typescript
'use client';

import { ComponentFromOriginalModule } from '@/components/emergency/EmergencyPage';
// Import the component that was used for this route in the original module

export default function EssentialPage() {
  return <ComponentFromOriginalModule />;
}
```

**Key points:**
- Add `'use client'` if the component uses hooks or client-side features
- Import the migrated component (you'll create these in Phase 6)
- Keep the page component thin - business logic stays in imported components

### 6. Create Layouts

If routes share common layouts, create `layout.tsx` files.

**Example: `src/app/essential/layout.tsx`**

```typescript
import { PropsWithChildren } from 'react';
import { EmergencyHeader } from '@/components/emergency/EmergencyHeader';

export default function EssentialLayout({ children }: PropsWithChildren) {
  return (
    <div>
      <EmergencyHeader />
      <main className="container mx-auto p-4">
        {children}
      </main>
    </div>
  );
}
```

**Layout hierarchy:**
- Root layout (`src/app/layout.tsx`) - Already created
- Section layouts (`src/app/essential/layout.tsx`) - Wraps all pages in that section
- Nested layouts - Can create deeper nesting as needed

### 7. Create Dynamic Routes

For routes with parameters (e.g., `:parkCode`, `:id`), use dynamic segments.

**Example: `src/app/essential/[parkCode]/[innerRoute]/page.tsx`**

```typescript
'use client';

import { use } from 'react';
import { MainComponent } from '@/components/emergency/Main';

interface PageProps {
  params: Promise<{
    parkCode: string;
    innerRoute: string;
  }>;
}

export default function EssentialDetailPage({ params }: PageProps) {
  const { parkCode, innerRoute } = use(params);

  return <MainComponent parkCode={parkCode} innerRoute={innerRoute} />;
}
```

**Key changes from React Router:**
- `useParams()` → Access via `params` prop
- Params are now a Promise and need to be unwrapped with `use()`
- Pass params as props to child components

### 8. Update Navigation Components

Find components that use React Router navigation and update them:

**Before (React Router v5):**
```typescript
import { useHistory, useLocation, Link } from 'react-router-dom';

function MyComponent() {
  const history = useHistory();
  const location = useLocation();

  const navigate = () => {
    history.push('/comms/emergency');
  };

  return <Link to="/comms/marketing">Marketing</Link>;
}
```

**After (Next.js):**
```typescript
'use client';

import { useRouter, usePathname } from 'next/navigation';
import Link from 'next/link';

function MyComponent() {
  const router = useRouter();
  const pathname = usePathname();

  const navigate = () => {
    router.push('/essential');
  };

  return <Link href="/marketing">Marketing</Link>;
}
```

**Migration map:**
- `useHistory()` → `useRouter()` from `next/navigation`
- `history.push(path)` → `router.push(path)`
- `history.replace(path)` → `router.replace(path)`
- `history.goBack()` → `router.back()`
- `useLocation()` → `usePathname()`, `useSearchParams()`
- `<Link to="">` → `<Link href="">`
- `<Redirect to="">` → `redirect()` function or `router.push()`

### 9. Handle Search Params and Query Strings

**Before (React Router):**
```typescript
import { useLocation } from 'react-router-dom';
import { parse } from 'query-string';

const location = useLocation();
const params = parse(location.search);
```

**After (Next.js):**
```typescript
'use client';

import { useSearchParams } from 'next/navigation';

const searchParams = useSearchParams();
const param = searchParams.get('key');
```

**For managing search params, use nuqs:**
```typescript
'use client';

import { useQueryState } from 'nuqs';

const [status, setStatus] = useQueryState('status');
// status comes from ?status=value
// setStatus('new-value') updates the URL
```

### 10. Create Route Groups (Optional)

If you want to organize routes without affecting URLs, use route groups with `(name)`:

```bash
src/app/
  (admin)/
    essential/
      page.tsx
    marketing/
      page.tsx
```

The `(admin)` group doesn't appear in URLs but helps with organization and shared layouts.

### 11. Update Routing in Provider

The `Providers` component already handles pathname changes for postMessage. Verify it works with your new routes:

```typescript
// In Providers.tsx - already implemented in Phase 4
useEffect(() => {
  const parts = pathname.split('/').filter(Boolean);
  if (parts[0] === 'embedded') {
    const appName = parts[1];
    const url = parts.slice(2).join('/');
    window.parent.postMessage(
      { type: 'pathChange', value: { appName, url } },
      '*'
    );
  }
}, [pathname]);
```

This notifies the parent application of route changes.

### 12. Test All Routes

Create a test navigation component to verify all routes work:

**Create `src/components/NavTest.tsx`:**
```typescript
'use client';

import Link from 'next/link';

export const NavTest = () => {
  return (
    <nav className="p-4 bg-gray-100 space-x-4">
      <Link href="/" className="underline">Home</Link>
      <Link href="/essential" className="underline">Essential</Link>
      <Link href="/marketing" className="underline">Marketing</Link>
    </nav>
  );
};
```

Add to pages temporarily for testing.

### 13. Document Route Mapping

Create a reference document showing the migration:

**Create `ROUTES.md` in your project:**
```markdown
# Route Migration Reference

## Original Routes (React Router v5)
- `/comms` → Redirected to `/comms/emergency`
- `/comms/emergency` → Emergency landing page
- `/comms/emergency/:parkCode/:innerRoute` → Park-specific emergency content
- `/comms/marketing` → Marketing landing page

## New Routes (Next.js App Router)
- `/` → Redirects to `/essential`
- `/essential` → Essential communications landing (was `/comms/emergency`)
- `/essential/[parkCode]/[innerRoute]` → Park-specific content
- `/marketing` → Marketing communications landing

## Embedded Routes
When embedded, all routes are prefixed with `/embedded/comms/`:
- `/embedded/comms/essential`
- `/embedded/comms/marketing`
```

### 14. Commit Routing Migration

```bash
git add .
git commit -m "feat: migrate routing to Next.js App Router

- Convert React Router v5 routes to App Router file structure
- Create page components for all main routes
- Implement layouts for shared UI structure
- Add dynamic route segments for parameters
- Update navigation components to use Next.js APIs
- Document route mapping

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

## Verification Checklist

Before moving to Phase 6, verify:

- [ ] All original routes mapped to Next.js structure
- [ ] Directory structure created in `src/app/`
- [ ] Root page.tsx created (landing or redirect)
- [ ] Section pages created
- [ ] Layouts created where needed
- [ ] Dynamic routes implemented for parameters
- [ ] Navigation components updated
- [ ] Search params handled with nuqs or useSearchParams
- [ ] Route mapping documented
- [ ] All routes accessible and render without errors
- [ ] Navigation between routes works
- [ ] Browser back/forward buttons work

## Test the Routing

```bash
npm run dev

# Visit each route:
http://localhost:3000
http://localhost:3000/essential
http://localhost:3000/marketing

# Test dynamic routes (may show placeholder content for now)
http://localhost:3000/essential/test-park/current

# Test navigation:
# - Click links between pages
# - Use browser back button
# - Check URL updates correctly
```

## Common Issues

**Issue: 404 on route that should exist**
- Verify `page.tsx` exists in the route directory
- Check file naming (must be exactly `page.tsx`, not `Page.tsx`)
- Ensure directory names match URL segments
- Restart dev server after creating new routes

**Issue: "use() is not a function" or params errors**
- In Next.js 15, params are Promises and must be unwrapped with `use()`
- Import `use` from React: `import { use } from 'react'`
- Or await params in async server components

**Issue: Layout not applying to pages**
- Verify `layout.tsx` is in the correct directory
- Check that layout exports a default function
- Ensure layout accepts and renders `children` prop

**Issue: "Cannot access pathname" or "useSearchParams is not defined"**
- These hooks require `'use client'` directive
- Import from `next/navigation`, not `next/router`
- Verify component is a client component

**Issue: Links not working**
- Use `Link` from `next/link`, not `react-router-dom`
- Use `href` prop, not `to`
- Ensure links use correct path (check against route structure)

**Issue: Redirect loops**
- Check root page.tsx doesn't redirect to itself
- Verify redirect logic conditions
- Use `router.replace()` instead of `router.push()` for redirects

## Next Steps

Once Phase 5 is complete and routing is working, proceed to:
**[Phase 6: Component Migration](./06-component-migration.md)**

In Phase 6, you'll migrate the actual business logic components that these routes render.

## Notes for Claude

When executing this phase:
1. **Thoroughly analyze source routing first** - understand all routes before creating structure
2. **Map routes logically** - embedded context may warrant different names
3. **Test each route** as you create it - don't wait until the end
4. **Document deviations** - if your module has unique routing needs, note them
5. **Keep pages thin** - business logic goes in components (migrated in Phase 6)
6. **Remember the embedded prefix** - routes work both standalone and at `/embedded/{app-name}/`
7. **Use dynamic routes sparingly** - only when truly needed for parameters
8. **Preserve route structure** - users may have bookmarked URLs (in embedded context)
