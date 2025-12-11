# Phase 6: Component Migration

## Overview

This phase migrates all business logic components, sections, and UI elements from the source module to the Next.js application. This is where the core functionality is preserved.

## Prerequisites

- [ ] Phase 1-5 completed
- [ ] Routing structure in place
- [ ] Source module components identified

## Step-by-Step Instructions

### 1. Inventory Source Components

Analyze the source module's component structure:

```bash
# In source module
tree src/components
tree src/sections
tree src/hooks
tree src/utils
```

**Create a migration checklist:**
- List all component files
- Note dependencies between components
- Identify which components are used by which routes
- Mark reusable vs route-specific components

### 2. Create Component Directory Structure

Organize components logically in the new app:

```bash
# Example structure
mkdir -p src/components/emergency
mkdir -p src/components/marketing
mkdir -p src/components/shared
mkdir -p src/components/ui
```

**Organization guidelines:**
- `src/components/shared/` - Reusable across sections
- `src/components/{section}/` - Section-specific components
- `src/components/ui/` - Generic UI components (buttons, modals, etc.)

### 3. Migrate Components One Section at a Time

Start with one section (e.g., emergency/essential):

**Migration steps for each component:**

1. **Copy the component file**
   ```bash
   cp ../app-haven-experience-admin-{name}/src/sections/Emergency/Component.tsx \
      src/components/emergency/Component.tsx
   ```

2. **Add client directive if needed**
   ```typescript
   'use client';
   // Add this at the top if component uses:
   // - useState, useEffect, or other React hooks
   // - Event handlers
   // - Browser APIs
   ```

3. **Update imports**
   - Change relative imports to use `@/` alias
   - Update `react-router-dom` imports to `next/navigation`
   - Fix any broken import paths

4. **Update routing hooks**
   ```typescript
   // Before
   import { useHistory, useParams, useLocation } from 'react-router-dom';

   // After
   import { useRouter, usePathname, useSearchParams } from 'next/navigation';
   ```

5. **Update navigation code**
   ```typescript
   // Before
   history.push('/path');

   // After
   router.push('/path');
   ```

6. **Update Link components**
   ```typescript
   // Before
   import { Link } from 'react-router-dom';
   <Link to="/path">Text</Link>

   // After
   import Link from 'next/link';
   <Link href="/path">Text</Link>
   ```

### 4. Migrate Styles

The source module likely uses SCSS modules. You have two options:

**Option A: Convert to Tailwind CSS (Recommended)**
```typescript
// Before
import classes from './Component.module.scss';
<div className={classes.container}>

// After
<div className="flex flex-col p-4 bg-white rounded shadow">
```

**Option B: Keep SCSS temporarily**
```typescript
// Keep imports as-is (SASS is installed)
import classes from './Component.module.scss';

// Copy SCSS file
cp ../app-haven-experience-admin-{name}/src/sections/Emergency/Component.module.scss \
   src/components/emergency/Component.module.scss
```

**Gradual migration approach:**
- Keep SCSS initially to get components working
- Convert to Tailwind incrementally
- Test thoroughly after each conversion

### 5. Handle SVG Assets

Copy SVG assets and update imports:

```bash
# Copy SVG files
mkdir -p src/assets
cp -r ../app-haven-experience-admin-{name}/src/assets/* src/assets/
```

**Update SVG imports:**
```typescript
// The webpack config (Phase 2) allows both:

// As React component
import PhoneIcon from '@/assets/phone.svg';
<PhoneIcon className="w-6 h-6" />

// As URL
import phoneUrl from '@/assets/phone.svg?url';
<img src={phoneUrl} alt="Phone" />
```

### 6. Migrate Provider Components

If the source has custom providers (like EmergencyFormProvider):

```bash
cp ../app-haven-experience-admin-{name}/src/providers/EmergencyFormProvider.tsx \
   src/providers/EmergencyFormProvider.tsx
```

Add `'use client'` and update imports. Then integrate into provider hierarchy if needed.

### 7. Migrate Custom Hooks

Copy hooks and update dependencies:

```bash
cp -r ../app-haven-experience-admin-{name}/src/hooks/* src/hooks/
```

**Update each hook:**
- Add `'use client'` if needed
- Update Router imports
- Update any API call patterns (covered in Phase 7)

### 8. Migrate Utility Functions

Copy utility functions:

```bash
cp -r ../app-haven-experience-admin-{name}/src/utils/* src/utils/
```

Most utilities won't need changes unless they depend on routing or environment.

### 9. Update Component Exports

Ensure components are exported correctly:

```typescript
// Prefer named exports
export const EmergencyPage = () => { ... };

// Or default export for page components
export default function EmergencyPage() { ... }
```

### 10. Connect Components to Routes

Update the page components created in Phase 5 to use migrated components:

```typescript
// src/app/essential/page.tsx
'use client';

import { EmergencyPage } from '@/components/emergency/EmergencyPage';

export default function EssentialPage() {
  return <EmergencyPage />;
}
```

### 11. Handle Forms and State Management

If components use forms:

**Controlled forms work the same:**
```typescript
'use client';

import { useState } from 'react';

export const MyForm = () => {
  const [value, setValue] = useState('');

  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      // handle submit
    }}>
      <input
        value={value}
        onChange={(e) => setValue(e.target.value)}
      />
    </form>
  );
};
```

**Consider using React Hook Form for complex forms:**
```bash
npm install react-hook-form
```

### 12. Handle Modals and Portals

If components use portals for modals:

```typescript
'use client';

import { createPortal } from 'react-dom';
import { useEffect, useState } from 'react';

export const Modal = ({ children, isOpen }: Props) => {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
    return () => setMounted(false);
  }, []);

  if (!mounted || !isOpen) return null;

  return createPortal(
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      {children}
    </div>,
    document.getElementById('react-portal')!
  );
};
```

The `react-portal` div was added to layout.tsx in Phase 4.

### 13. Test Each Component

As you migrate components, test them:

```bash
npm run dev
# Navigate to routes using the component
# Verify functionality
# Check console for errors
```

### 14. Handle Shared Package Components

If using `@havenengineering/experience-admin-styleguide`:

```typescript
// These should work as-is
import Button from '@havenengineering/experience-admin-styleguide/components/Button';
import CircularProgress from '@havenengineering/experience-admin-styleguide/components/CircularProgress';
```

No migration needed for shared package components.

### 15. Document Component Changes

Keep notes on significant changes:

**Create `MIGRATION_NOTES.md`:**
```markdown
# Component Migration Notes

## EmergencyPage
- Converted useHistory to useRouter
- Changed SCSS to Tailwind classes
- Updated form submission to use React Query mutation

## MarketingHeader
- Kept SCSS modules temporarily
- Navigation links updated for Next.js

## Changes to Review
- All forms now use controlled components
- Modal rendering uses portal pattern
```

### 16. Commit Component Migration

Commit frequently as you migrate component groups:

```bash
git add .
git commit -m "feat: migrate emergency/essential components

- Migrate EmergencyPage, EmergencyHeader components
- Convert routing hooks to Next.js equivalents
- Update navigation and Link components
- Copy and integrate SVG assets
- Preserve all business logic

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

## Verification Checklist

Before moving to Phase 7, verify:

- [ ] All components copied from source module
- [ ] Client directives added where needed
- [ ] All imports updated to use @/ alias
- [ ] Routing hooks updated to Next.js equivalents
- [ ] Link components migrated to next/link
- [ ] SVG assets copied and imports updated
- [ ] Styles migrated or SCSS modules copied
- [ ] Custom hooks migrated and working
- [ ] Utility functions copied
- [ ] Components connected to routes
- [ ] All pages render without errors
- [ ] Navigation between pages works
- [ ] Forms and interactions work
- [ ] No console errors or warnings

## Testing Strategy

Test systematically:

1. **Visual testing** - Each page should render correctly
2. **Interaction testing** - Forms, buttons, clicks work
3. **Navigation testing** - Links go to correct pages
4. **Data testing** - Components receive correct props
5. **Error testing** - Error states display properly

## Common Issues

**Issue: "Text content does not match server-rendered HTML"**
- Hydration mismatch between server and client
- Ensure client-only code is in useEffect
- Use `'use client'` directive

**Issue: Component renders twice**
- React 18 Strict Mode behavior in development
- Expected behavior, won't happen in production

**Issue: "Cannot read property of undefined"**
- Check all optional chaining (?.) is in place
- Verify data dependencies in useEffect
- Ensure props are passed correctly

**Issue: Styles not applying**
- Verify Tailwind classes are in content config
- Check SCSS module imports have correct path
- Restart dev server after CSS changes

**Issue: SVG not rendering**
- Check webpack config includes @svgr/webpack (Phase 2)
- Verify import path is correct
- Check `global.d.ts` has SVG type declarations

## Next Steps

Once Phase 6 is complete and components are migrated, proceed to:
**[Phase 7: API & Data Fetching](./07-api-data-fetching.md)**

## Notes for Claude

When executing this phase:
1. **Migrate incrementally** - One section at a time
2. **Test after each component** - Don't wait until everything is migrated
3. **Preserve business logic exactly** - Don't refactor unless necessary
4. **Document changes** - Note any deviations or challenges
5. **Keep commits focused** - Commit by section or feature group
6. **Watch for router dependencies** - Most common migration issue
7. **Verify all user interactions** - Forms, clicks, navigation
8. **Check error handling** - Ensure error states still work
