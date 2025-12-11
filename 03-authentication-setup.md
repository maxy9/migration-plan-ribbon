# Phase 3: Authentication Setup

## Overview

This phase implements Microsoft Authentication Library (MSAL) integration for Azure AD authentication. All Ribbon apps use the same MSAL configuration pattern for consistency.

## Prerequisites

- [ ] Phase 1 and 2 completed
- [ ] `@azure/msal-react` installed
- [ ] Environment variables configured

## Step-by-Step Instructions

### 1. Create MSAL Configuration

Create `src/msal/msalConfig.ts`:

```typescript
export const msalConfig = {
  auth: {
    clientId: process.env.NEXT_PUBLIC_AZURE_AD_CLIENT_ID as string,
    authority: `https://login.microsoftonline.com/${process.env.NEXT_PUBLIC_AZURE_AD_TENANT_ID as string}`,
    redirectUri: '/',
  },
  cache: {
    storeAuthStateInCookie: false,
    cacheLocation: 'localStorage' as const,
  },
  system: {
    allowRedirectInIframe: true,
  },
};

export const loginRequest = {
  scopes: ['User.Read'],
  refreshTokenExpirationOffsetSeconds: 1800,
};
```

**Key configuration points:**
- `clientId` and `authority` from environment variables
- `redirectUri` set to `/` (root of the embedded app)
- `allowRedirectInIframe: true` - Required for embedded iframe authentication
- `cacheLocation: 'localStorage'` - Persist auth across page reloads
- Login scopes include `User.Read` for basic user profile

### 2. Create MSAL Instance

Create `src/msal/MsalInstance.ts`:

```typescript
'use client';

import { PublicClientApplication } from '@azure/msal-browser';
import { msalConfig } from './msalConfig';

// Create singleton MSAL instance
let msalInstance: PublicClientApplication | null = null;

export const getMsalInstance = (): PublicClientApplication => {
  if (!msalInstance) {
    msalInstance = new PublicClientApplication(msalConfig);
  }
  return msalInstance;
};

// Export the instance for direct use
export const msalInstance = getMsalInstance();
```

**Why a singleton?**
- MSAL instance must be shared across the entire application
- Prevents multiple authentication popup/redirect flows
- Maintains consistent auth state

### 3. Create Session Provider

Create `src/providers/SessionProvider.tsx`:

```typescript
'use client';

import { PropsWithChildren, useEffect, useState } from 'react';
import { useMsal } from '@azure/msal-react';
import { InteractionStatus } from '@azure/msal-browser';
import { loginRequest } from '@/msal/msalConfig';

interface SessionProviderProps extends PropsWithChildren {
  sid?: string;
}

export const SessionProvider = ({ children, sid }: SessionProviderProps) => {
  const { instance, inProgress } = useMsal();
  const [isInitialized, setIsInitialized] = useState(false);

  useEffect(() => {
    if (inProgress === InteractionStatus.None && !isInitialized) {
      const initAuth = async () => {
        try {
          // Handle redirect promise
          await instance.handleRedirectPromise();

          const accounts = instance.getAllAccounts();

          if (accounts.length === 0) {
            // No authenticated user, trigger login
            if (sid) {
              // If session ID provided, use redirect (embedded scenario)
              await instance.loginRedirect({
                ...loginRequest,
                prompt: 'select_account',
              });
            } else {
              // Otherwise use popup
              await instance.loginPopup(loginRequest);
            }
          } else {
            // Set active account
            instance.setActiveAccount(accounts[0]);
          }

          setIsInitialized(true);
        } catch (error) {
          console.error('Authentication error:', error);
        }
      };

      initAuth();
    }
  }, [instance, inProgress, isInitialized, sid]);

  // Don't render children until auth is initialized
  if (!isInitialized) {
    return <div>Initializing authentication...</div>;
  }

  return <>{children}</>;
};
```

**Key functionality:**
- Handles redirect promise from authentication flows
- Triggers login if no authenticated user
- Uses redirect for embedded scenarios (with `sid`), popup otherwise
- Sets active account for use throughout the app
- Shows loading state during initialization

### 4. Create Auth Provider

Create `src/providers/AuthProvider.tsx`:

```typescript
'use client';

import {
  AuthenticatedTemplate,
  MsalProvider,
  UnauthenticatedTemplate
} from '@azure/msal-react';
import { SessionProvider } from '@/providers/SessionProvider';
import { msalInstance } from '@/msal/MsalInstance';
import { PropsWithChildren } from 'react';

interface AuthProviderProps extends PropsWithChildren {
  sid?: string;
}

export const AuthProvider = ({ children, sid }: AuthProviderProps) => {
  return (
    <MsalProvider instance={msalInstance}>
      <SessionProvider sid={sid}>
        <AuthenticatedTemplate>
          {children}
        </AuthenticatedTemplate>
        <UnauthenticatedTemplate>
          <div>Authenticating...</div>
        </UnauthenticatedTemplate>
      </SessionProvider>
    </MsalProvider>
  );
};
```

**Component hierarchy:**
- `MsalProvider`: Provides MSAL context to all children
- `SessionProvider`: Handles authentication initialization
- `AuthenticatedTemplate`: Renders children only when authenticated
- `UnauthenticatedTemplate`: Shows loading state when not authenticated

### 5. Create Logout Component (Optional)

Create `src/components/Logout.tsx` for testing:

```typescript
'use client';

import { useMsal } from '@azure/msal-react';

export const Logout = () => {
  const { instance } = useMsal();

  const handleLogout = () => {
    instance.logoutPopup({
      mainWindowRedirectUri: '/',
    });
  };

  return (
    <button
      onClick={handleLogout}
      className="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
    >
      Logout
    </button>
  );
};
```

This is useful for testing authentication during development.

### 6. Create useAuth Hook

Create `src/hooks/useAuth.ts` for easy access to auth state:

```typescript
'use client';

import { useMsal } from '@azure/msal-react';
import { AccountInfo } from '@azure/msal-browser';

export const useAuth = () => {
  const { instance, accounts } = useMsal();

  const account: AccountInfo | null = accounts[0] || null;

  const getAccessToken = async (scopes: string[] = ['User.Read']): Promise<string> => {
    if (!account) {
      throw new Error('No active account');
    }

    try {
      const response = await instance.acquireTokenSilent({
        scopes,
        account,
      });
      return response.accessToken;
    } catch (error) {
      // If silent token acquisition fails, try interactive
      const response = await instance.acquireTokenPopup({
        scopes,
        account,
      });
      return response.accessToken;
    }
  };

  return {
    account,
    user: account?.name || null,
    email: account?.username || null,
    isAuthenticated: !!account,
    getAccessToken,
  };
};
```

**Usage in components:**
```typescript
const { user, email, isAuthenticated, getAccessToken } = useAuth();
```

### 7. Update Environment Variables

Verify `.env.production` has MSAL credentials:

```bash
#MSAL
NEXT_PUBLIC_AZURE_AD_CLIENT_ID=75de59c0-2a7f-4d4a-baa6-fcc8987e1615
NEXT_PUBLIC_AZURE_AD_TENANT_ID=6bec8931-53ed-4238-a6a8-495fa78471d5
```

### 8. Test Authentication Setup

Update `src/app/page.tsx` temporarily to test auth:

```typescript
'use client';

import { useAuth } from '@/hooks/useAuth';
import { Logout } from '@/components/Logout';

export default function Home() {
  const { user, email, isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <div>Not authenticated</div>;
  }

  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-4">Authentication Test</h1>
      <div className="space-y-2">
        <p>User: {user}</p>
        <p>Email: {email}</p>
        <Logout />
      </div>
    </div>
  );
}
```

**Note:** This will be replaced in Phase 5 when we implement proper routing.

### 9. Commit Authentication Setup

```bash
git add .
git commit -m "feat: implement MSAL authentication

- Create MSAL configuration and instance
- Implement SessionProvider for auth initialization
- Create AuthProvider with authenticated templates
- Add useAuth hook for easy auth state access
- Create Logout component for testing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

## Verification Checklist

Before moving to Phase 4, verify:

- [ ] `src/msal/msalConfig.ts` created with proper configuration
- [ ] `src/msal/MsalInstance.ts` created with singleton pattern
- [ ] `src/providers/SessionProvider.tsx` implemented
- [ ] `src/providers/AuthProvider.tsx` implemented
- [ ] `src/hooks/useAuth.ts` created
- [ ] `src/components/Logout.tsx` created
- [ ] Environment variables set correctly
- [ ] App runs and triggers authentication flow
- [ ] After authentication, user info displays correctly
- [ ] Logout button works

## Test the Authentication

```bash
# Start dev server
npm run dev

# Visit http://localhost:3000
# Should see authentication flow:
# 1. "Initializing authentication..."
# 2. Login popup or redirect
# 3. After login: User info displays
# 4. Logout button works
```

## Common Issues

**Issue: "AADB2C90077: User does not have an existing session"**
- This is normal for first-time authentication
- User needs to complete the login flow

**Issue: "Redirect URI mismatch"**
- Verify `NEXT_PUBLIC_AZURE_AD_CLIENT_ID` matches Azure AD app configuration
- Check that redirect URI is registered in Azure AD

**Issue: Authentication works but user info is null**
- Check that `User.Read` scope is granted
- Verify account is being set correctly in SessionProvider

**Issue: "Instance initialization failed"**
- Ensure environment variables are prefixed with `NEXT_PUBLIC_`
- Verify variables are loaded (check browser console)

**Issue: Popup blocked**
- Browser may block authentication popup
- Instruct user to allow popups or use redirect flow

## Next Steps

Once Phase 3 is complete and authentication is working, proceed to:
**[Phase 4: Provider Architecture](./04-provider-architecture.md)**

## Notes for Claude

When executing this phase:
1. Copy the code exactly as shown - MSAL configuration is standardized
2. Test authentication before moving to Phase 4
3. Make sure `'use client'` directives are present (MSAL requires client-side)
4. Don't modify MSAL configuration unless specifically needed
5. Verify environment variables are accessible in the browser
