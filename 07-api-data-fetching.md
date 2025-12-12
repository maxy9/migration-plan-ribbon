# Phase 7: API & Data Fetching

## Overview

This phase migrates API client code and data fetching logic to use React Query (TanStack Query) for efficient data management, caching, and state handling.

## Prerequisites

- [ ] Phase 1-6 completed
- [ ] Components migrated
- [ ] React Query provider set up (Phase 4)

## Step-by-Step Instructions

### 1. Analyze Source API Code

Identify API patterns in the source module:

```bash
# In source module
ls src/api/
cat src/api/*.ts

# Find where API calls are made
grep -r "fetch\|axios\|api\." src/
```

**Document:**
- What API endpoints are called?
- What authentication is required?
- What data transformations happen?
- Are there any caching strategies?

### 2. Create API Client

Create `src/lib/api/client.ts`:

```typescript
import axios, { AxiosInstance } from 'axios';

// Create axios instance
export const apiClient: AxiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL || '/api',
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add request interceptor for authentication
apiClient.interceptors.request.use(
  async (config) => {
    // Add auth token if available
    // This will be integrated with MSAL in next step
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Add response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    // Handle common errors
    if (error.response?.status === 401) {
      // Handle unauthorized
      console.error('Unauthorized request');
    }
    return Promise.reject(error);
  }
);
```

### 3. Integrate MSAL Authentication with API Client

Create `src/lib/api/auth.ts`:

```typescript
import { apiClient } from './client';
import { msalInstance } from '@/msal/MsalInstance';
import { loginRequest } from '@/msal/msalConfig';

export const addAuthToken = async () => {
  try {
    const accounts = msalInstance.getAllAccounts();
    if (accounts.length === 0) {
      throw new Error('No authenticated account');
    }

    const response = await msalInstance.acquireTokenSilent({
      ...loginRequest,
      account: accounts[0],
    });

    return response.accessToken;
  } catch (error) {
    console.error('Failed to acquire token:', error);
    throw error;
  }
};

// Configure interceptor to add token
apiClient.interceptors.request.use(
  async (config) => {
    try {
      const token = await addAuthToken();
      config.headers.Authorization = `Bearer ${token}`;
    } catch (error) {
      console.error('Auth token error:', error);
    }
    return config;
  },
  (error) => Promise.reject(error)
);
```

### 4. Create API Service Functions

Organize API calls by domain. Create `src/lib/api/comms.ts`:

```typescript
import { apiClient } from './client';

export interface EmergencyComm {
  id: string;
  title: string;
  message: string;
  createdAt: string;
  // ... other fields
}

export const commsApi = {
  // Get emergency communications
  getEmergencyComms: async (parkId: string): Promise<EmergencyComm[]> => {
    const { data } = await apiClient.get(`/parks/${parkId}/comms/emergency`);
    return data;
  },

  // Create emergency communication
  createEmergencyComm: async (
    parkId: string,
    comm: Partial<EmergencyComm>
  ): Promise<EmergencyComm> => {
    const { data } = await apiClient.post(
      `/parks/${parkId}/comms/emergency`,
      comm
    );
    return data;
  },

  // Update emergency communication
  updateEmergencyComm: async (
    parkId: string,
    commId: string,
    updates: Partial<EmergencyComm>
  ): Promise<EmergencyComm> => {
    const { data } = await apiClient.patch(
      `/parks/${parkId}/comms/emergency/${commId}`,
      updates
    );
    return data;
  },

  // Delete emergency communication
  deleteEmergencyComm: async (
    parkId: string,
    commId: string
  ): Promise<void> => {
    await apiClient.delete(`/parks/${parkId}/comms/emergency/${commId}`);
  },
};
```

**Key points:**
- Group related endpoints
- Type all request and response data
- Keep functions pure (no side effects)
- Return data directly, not full response

### 5. Create Query Hooks

Create `src/hooks/useEmergencyComms.ts`:

```typescript
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { commsApi, EmergencyComm } from '@/lib/api/comms';
import { usePark } from '@/providers/ParkProvider';

// Query keys
export const emergencyCommsKeys = {
  all: ['emergency-comms'] as const,
  lists: () => [...emergencyCommsKeys.all, 'list'] as const,
  list: (parkId: string) => [...emergencyCommsKeys.lists(), parkId] as const,
  details: () => [...emergencyCommsKeys.all, 'detail'] as const,
  detail: (id: string) => [...emergencyCommsKeys.details(), id] as const,
};

// Get emergency communications
export const useEmergencyComms = () => {
  const { park } = usePark();

  return useQuery({
    queryKey: emergencyCommsKeys.list(park?.id || ''),
    queryFn: () => {
      if (!park?.id) throw new Error('Park not available');
      return commsApi.getEmergencyComms(park.id);
    },
    enabled: !!park?.id,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};

// Create emergency communication
export const useCreateEmergencyComm = () => {
  const { park } = usePark();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (comm: Partial<EmergencyComm>) => {
      if (!park?.id) throw new Error('Park not available');
      return commsApi.createEmergencyComm(park.id, comm);
    },
    onSuccess: () => {
      // Invalidate and refetch
      queryClient.invalidateQueries({
        queryKey: emergencyCommsKeys.lists(),
      });
    },
  });
};

// Update emergency communication
export const useUpdateEmergencyComm = () => {
  const { park } = usePark();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ commId, updates }: { commId: string; updates: Partial<EmergencyComm> }) => {
      if (!park?.id) throw new Error('Park not available');
      return commsApi.updateEmergencyComm(park.id, commId, updates);
    },
    onSuccess: (_, { commId }) => {
      queryClient.invalidateQueries({
        queryKey: emergencyCommsKeys.list(park?.id || ''),
      });
      queryClient.invalidateQueries({
        queryKey: emergencyCommsKeys.detail(commId),
      });
    },
  });
};

// Delete emergency communication
export const useDeleteEmergencyComm = () => {
  const { park } = usePark();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (commId: string) => {
      if (!park?.id) throw new Error('Park not available');
      return commsApi.deleteEmergencyComm(park.id, commId);
    },
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: emergencyCommsKeys.lists(),
      });
    },
  });
};
```

**React Query patterns:**
- **Queries** - For fetching data (GET requests)
- **Mutations** - For modifying data (POST, PUT, PATCH, DELETE)
- **Query Keys** - Hierarchical keys for caching and invalidation
- **Invalidation** - Refetch after mutations to keep UI in sync

### 6. Use Query Hooks in Components

Update components to use the new hooks:

```typescript
'use client';

import { useEmergencyComms, useCreateEmergencyComm } from '@/hooks/useEmergencyComms';

export const EmergencyPage = () => {
  const { data: comms, isLoading, error } = useEmergencyComms();
  const createComm = useCreateEmergencyComm();

  const handleCreate = async (newComm: Partial<EmergencyComm>) => {
    try {
      await createComm.mutateAsync(newComm);
      // Success!
    } catch (error) {
      console.error('Failed to create:', error);
    }
  };

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {comms?.map((comm) => (
        <div key={comm.id}>{comm.title}</div>
      ))}
      <button onClick={() => handleCreate({ title: 'Test' })}>
        Create
      </button>
    </div>
  );
};
```

### 7. Handle Loading and Error States

Create reusable loading/error components:

**Create `src/components/shared/LoadingSpinner.tsx`:**
```typescript
export const LoadingSpinner = () => (
  <div className="flex justify-center items-center p-8">
    <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900" />
  </div>
);
```

**Create `src/components/shared/ErrorMessage.tsx`:**
```typescript
interface ErrorMessageProps {
  message: string;
  onRetry?: () => void;
}

export const ErrorMessage = ({ message, onRetry }: ErrorMessageProps) => (
  <div className="bg-red-50 border border-red-200 rounded p-4">
    <p className="text-red-800">{message}</p>
    {onRetry && (
      <button
        onClick={onRetry}
        className="mt-2 px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
      >
        Retry
      </button>
    )}
  </div>
);
```

### 8. Add Environment Variables for API

Update `.env.production`:

```bash
#MSAL
NEXT_PUBLIC_AZURE_AD_CLIENT_ID=75de59c0-2a7f-4d4a-baa6-fcc8987e1615
NEXT_PUBLIC_AZURE_AD_TENANT_ID=6bec8931-53ed-4238-a6a8-495fa78471d5

# API Configuration
NEXT_PUBLIC_API_BASE_URL=https://api.haven.com
```

Update `.env.example` accordingly.

### 9. Handle Optimistic Updates (Optional)

For better UX, implement optimistic updates:

```typescript
export const useUpdateEmergencyComm = () => {
  const { park } = usePark();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ commId, updates }: { commId: string; updates: Partial<EmergencyComm> }) => {
      if (!park?.id) throw new Error('Park not available');
      return commsApi.updateEmergencyComm(park.id, commId, updates);
    },
    onMutate: async ({ commId, updates }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({
        queryKey: emergencyCommsKeys.list(park?.id || ''),
      });

      // Snapshot previous value
      const previousComms = queryClient.getQueryData(
        emergencyCommsKeys.list(park?.id || '')
      );

      // Optimistically update
      queryClient.setQueryData(
        emergencyCommsKeys.list(park?.id || ''),
        (old: EmergencyComm[] | undefined) =>
          old?.map((comm) =>
            comm.id === commId ? { ...comm, ...updates } : comm
          )
      );

      return { previousComms };
    },
    onError: (err, variables, context) => {
      // Rollback on error
      if (context?.previousComms) {
        queryClient.setQueryData(
          emergencyCommsKeys.list(park?.id || ''),
          context.previousComms
        );
      }
    },
    onSettled: () => {
      // Refetch after mutation
      queryClient.invalidateQueries({
        queryKey: emergencyCommsKeys.lists(),
      });
    },
  });
};
```

### 10. Migrate Existing API Calls

Find and replace direct API calls in components:

**Before:**
```typescript
useEffect(() => {
  fetch('/api/comms')
    .then((res) => res.json())
    .then((data) => setComms(data));
}, []);
```

**After:**
```typescript
const { data: comms } = useEmergencyComms();
```

### 11. Test Data Fetching

```bash
npm run dev

# Test:
# - Data loads on page mount
# - Loading states display
# - Error states display correctly
# - Mutations update data
# - Caching works (no re-fetch on navigation back)
```

### 12. API & Data Fetching Complete

All API integration and data fetching is set up. Continue to Phase 8 for Docker and deployment configuration.

## Verification Checklist

- [ ] API client created and configured
- [ ] MSAL authentication integrated with API calls
- [ ] API service functions created for all endpoints
- [ ] Query hooks created for data fetching
- [ ] Mutation hooks created for data updates
- [ ] Components updated to use hooks
- [ ] Loading states implemented
- [ ] Error handling implemented
- [ ] Environment variables configured
- [ ] All API calls working
- [ ] Data caching works correctly
- [ ] Mutations invalidate and refetch appropriately

## Common Issues

**Issue: API calls fail with 401 Unauthorized**
- Verify MSAL token is being added to headers
- Check token acquisition is working
- Ensure API accepts Bearer token format

**Issue: Data not refetching after mutation**
- Check query invalidation is called in mutation onSuccess
- Verify query keys match between query and invalidation
- Use React Query DevTools to debug cache

**Issue: Infinite refetch loop**
- Check enabled condition on queries
- Verify dependencies in query keys don't change unnecessarily
- Review staleTime configuration

**Issue: TypeScript errors on API responses**
- Type all API response interfaces
- Ensure API service functions have correct return types
- Use type guards for runtime checking if needed

## Next Steps

**[Phase 8: Docker & Deployment](./08-docker-deployment.md)**

## Notes for Claude

1. Preserve all business logic from original API calls
2. Test each endpoint after migration
3. Use React Query DevTools to debug caching
4. Keep API functions pure and testable
5. Type all API data for safety
