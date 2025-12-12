# Phase 4: Provider Architecture

## Overview

This phase sets up the provider hierarchy that wraps the entire application, including React Query for data fetching, Park context for park-specific data, and postMessage communication with the parent application.

## Prerequisites

- [ ] Phase 1, 2, and 3 completed
- [ ] Authentication working
- [ ] Dependencies installed

## Step-by-Step Instructions

### 1. Create Query Provider

Create `src/providers/QueryProvider.tsx`:

```typescript
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { PropsWithChildren, useState } from 'react';

export const QueryProvider = ({ children }: PropsWithChildren) => {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
            retry: 1,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
};
```

**Configuration choices:**
- `staleTime: 60 * 1000` - Data considered fresh for 1 minute
- `refetchOnWindowFocus: false` - Don't refetch when window regains focus (useful in iframes)
- `retry: 1` - Only retry failed requests once
- DevTools only included in development builds

### 2. Create Park Provider

Create `src/providers/ParkProvider.tsx`:

```typescript
'use client';

import { createContext, PropsWithChildren, useContext, useEffect, useState } from 'react';

interface Park {
  id: string;
  name: string;
  // Add other park properties as needed
}

interface ParkContextValue {
  park: Park | null;
  isLoading: boolean;
  setPark: (park: Park | null) => void;
}

const ParkContext = createContext<ParkContextValue | undefined>(undefined);

interface ParkProviderProps extends PropsWithChildren {
  waitForPark?: boolean;
}

export const ParkProvider = ({ children, waitForPark = false }: ParkProviderProps) => {
  const [park, setPark] = useState<Park | null>(null);
  const [isLoading, setIsLoading] = useState(waitForPark);

  useEffect(() => {
    // Listen for park data from parent
    const handleMessage = (event: MessageEvent) => {
      if (event.data.type === 'setPark') {
        setPark(event.data.value);
        setIsLoading(false);
      }
    };

    window.addEventListener('message', handleMessage);

    // Request park data from parent
    window.parent.postMessage({ type: 'getPark' }, '*');

    return () => {
      window.removeEventListener('message', handleMessage);
    };
  }, []);

  // If waiting for park and still loading, don't render children
  if (waitForPark && isLoading) {
    return <div>Loading park data...</div>;
  }

  return (
    <ParkContext.Provider value={{ park, isLoading, setPark }}>
      {children}
    </ParkContext.Provider>
  );
};

export const usePark = () => {
  const context = useContext(ParkContext);
  if (context === undefined) {
    throw new Error('usePark must be used within a ParkProvider');
  }
  return context;
};
```

**Key functionality:**
- Listens for `setPark` messages from parent application
- Requests park data on mount
- `waitForPark` prop prevents rendering until park data received
- Provides `usePark()` hook for accessing park context

### 3. Create Root Providers Component

Create `src/providers/Providers.tsx`:

```typescript
'use client';

import { PropsWithChildren, Suspense, useCallback, useEffect } from 'react';
import { QueryProvider } from '@/providers/QueryProvider';
import { AuthProvider } from '@/providers/AuthProvider';
import { ParkProvider } from '@/providers/ParkProvider';
import { usePathname, useRouter, useSearchParams } from 'next/navigation';

export const ProvidersWrapped = ({ children }: PropsWithChildren) => {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const { push } = useRouter();
  const sid = searchParams.get('sid') || undefined;

  // Notify parent of parameter changes
  useEffect(() => {
    window.parent.postMessage(
      { type: 'paramChange', value: searchParams.toString() },
      '*'
    );
  }, [searchParams]);

  // Notify parent of path changes
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

  // Listen for navigation messages from parent
  const handleMessage = useCallback(
    (event: MessageEvent) => {
      if (event.data.type === 'navigate') {
        push(event.data.value);
      }
    },
    [push]
  );

  useEffect(() => {
    window.addEventListener('message', handleMessage);
    return () => {
      window.removeEventListener('message', handleMessage);
    };
  }, [handleMessage]);

  return (
    <QueryProvider>
      <AuthProvider sid={sid}>
        <ParkProvider waitForPark={!!sid}>{children}</ParkProvider>
      </AuthProvider>
    </QueryProvider>
  );
};

export const Providers = ({ children }: PropsWithChildren) => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <ProvidersWrapped>{children}</ProvidersWrapped>
    </Suspense>
  );
};
```

**Provider hierarchy:**
1. `Suspense` - Handles loading states for async components
2. `QueryProvider` - React Query context
3. `AuthProvider` - MSAL authentication
4. `ParkProvider` - Park context data

**PostMessage API:**
- **Sends to parent:**
  - `paramChange` - When URL search params change
  - `pathChange` - When route path changes
- **Receives from parent:**
  - `navigate` - Navigate to a specific route
  - `setPark` - Set park data (handled by ParkProvider)

### 4. Update Root Layout

Update `src/app/layout.tsx` to use the Providers:

```typescript
import type { Metadata } from 'next';
import { Roboto, Open_Sans } from 'next/font/google';
import './globals.css';
import { NuqsAdapter } from 'nuqs/adapters/next/app';
import { Providers } from '@/providers/Providers';

const roboto = Roboto({
  variable: '--font-roboto',
  subsets: ['latin'],
  weight: ['400', '500', '700'],
});

const openSans = Open_Sans({
  variable: '--font-open-sans',
  subsets: ['latin'],
});

export const metadata: Metadata = {
  title: 'Haven Ribbon App',
  description: 'Haven Experience Admin Application',
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className={`${roboto.variable} ${openSans.variable} antialiased`}>
        <Providers>
          <NuqsAdapter>{children}</NuqsAdapter>
        </Providers>
        <div id="react-portal" />
      </body>
    </html>
  );
}
```

**Key additions:**
- `Providers` wrapper with all context providers
- `NuqsAdapter` for URL state management
- `react-portal` div for modals/dialogs
- Font variables for consistent typography

### 5. Create useParkQuery Hook (Optional but Recommended)

Create `src/hooks/useParkQuery.ts` for park-aware queries:

```typescript
'use client';

import { useQuery, UseQueryOptions } from '@tanstack/react-query';
import { usePark } from '@/providers/ParkProvider';

export const useParkQuery = <TData, TError = Error>(
  queryKey: unknown[],
  queryFn: (parkId: string) => Promise<TData>,
  options?: Omit<UseQueryOptions<TData, TError>, 'queryKey' | 'queryFn'>
) => {
  const { park } = usePark();

  return useQuery<TData, TError>({
    queryKey: ['park', park?.id, ...queryKey],
    queryFn: () => {
      if (!park?.id) {
        throw new Error('Park not available');
      }
      return queryFn(park.id);
    },
    enabled: !!park?.id && (options?.enabled !== false),
    ...options,
  });
};
```

**Usage:**
```typescript
const { data, isLoading } = useParkQuery(
  ['comms', 'emergency'],
  (parkId) => fetchEmergencyComms(parkId)
);
```

This ensures queries include park ID and wait for park data before executing.

### 6. Create Test Page

Update `src/app/page.tsx` to test the provider setup:

```typescript
'use client';

import { useAuth } from '@/hooks/useAuth';
import { usePark } from '@/providers/ParkProvider';
import { Logout } from '@/components/Logout';

export default function Home() {
  const { user, email } = useAuth();
  const { park, isLoading: parkLoading } = usePark();

  return (
    <div className="p-8 space-y-4">
      <h1 className="text-2xl font-bold">Provider Test</h1>

      <div className="border p-4 rounded">
        <h2 className="font-bold mb-2">Authentication</h2>
        <p>User: {user}</p>
        <p>Email: {email}</p>
        <Logout />
      </div>

      <div className="border p-4 rounded">
        <h2 className="font-bold mb-2">Park Context</h2>
        {parkLoading ? (
          <p>Loading park...</p>
        ) : park ? (
          <>
            <p>Park ID: {park.id}</p>
            <p>Park Name: {park.name}</p>
          </>
        ) : (
          <p>No park data (normal when not embedded)</p>
        )}
      </div>
    </div>
  );
}
```

### 7. Provider Architecture Complete

All providers are now set up. Continue to Phase 5 to migrate routing.

## Verification Checklist

Before moving to Phase 5, verify:

- [ ] `QueryProvider` created with React Query setup
- [ ] `ParkProvider` created with context and postMessage handling
- [ ] `Providers` component created with full hierarchy
- [ ] Root layout updated to use Providers
- [ ] `useParkQuery` hook created (optional)
- [ ] Test page shows auth and park data
- [ ] React Query DevTools visible in dev mode
- [ ] No console errors on page load

## Test the Providers

```bash
# Start dev server
npm run dev

# Visit http://localhost:3000
# Should see:
# - Authentication info
# - Park context (will show "No park data" when not embedded - this is normal)
# - No console errors
```

**To test embedded with park data**, you'll need to create a test parent page that sends postMessage events. This can be done later when testing the full integration.

## Common Issues

**Issue: "usePark must be used within a ParkProvider"**
- Ensure `ParkProvider` is in the Providers hierarchy
- Verify component using `usePark()` is a child of Providers

**Issue: Search params not updating**
- Check that `NuqsAdapter` wraps the children in layout
- Verify `useSearchParams()` is only used in client components

**Issue: "Cannot read property 'postMessage' of undefined"**
- This happens in non-browser environments (SSR)
- Ensure postMessage code is in `useEffect` hooks
- Add `'use client'` directive to components using postMessage

**Issue: Park data never loads**
- Normal when running standalone (not embedded)
- To test park functionality, you'll need a parent application or test harness

**Issue: React Query not working**
- Verify `@tanstack/react-query` is installed
- Check that QueryProvider is mounted before components using queries
- Ensure queries are called in client components

## Next Steps

Once Phase 4 is complete and providers are working, proceed to:
**[Phase 5: Routing Migration](./05-routing-migration.md)**

## Notes for Claude

When executing this phase:
1. Verify all providers render without errors
2. Test authentication and park context display
3. Check browser console for postMessage events
4. Ensure React Query DevTools appear in development
5. Don't worry if park data is null when running standalone - this is expected
