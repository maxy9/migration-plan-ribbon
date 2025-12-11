# Phase 2: Configuration Files

## Overview

This phase sets up all the configuration files needed for the NextJS application, including Next.js config, TypeScript, Tailwind CSS, ESLint, Prettier, and environment variables.

## Prerequisites

- [ ] Phase 1 completed
- [ ] Repository initialized with Next.js

## Step-by-Step Instructions

### 1. Configure Next.js

Replace the content of `next.config.ts` (or create it if using `.js`):

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
  webpack(config) {
    // SVG handling configuration
    const fileLoaderRule = config.module.rules.find(
      (rule: { test: { test: (arg0: string) => never } }) => rule.test?.test?.('.svg')
    );

    config.module.rules.push(
      {
        ...fileLoaderRule,
        test: /\.svg$/i,
        resourceQuery: /url/, // *.svg?url
      },
      {
        test: /\.svg$/i,
        issuer: fileLoaderRule.issuer,
        resourceQuery: { not: [...fileLoaderRule.resourceQuery.not, /url/] }, // exclude if *.svg?url
        use: ['@svgr/webpack'],
      }
    );

    fileLoaderRule.exclude = /\.svg$/i;

    return config;
  },
};

export default nextConfig;
```

**Replace `{app-name}` with your actual app name** (e.g., `comms`, `venues`, etc.)

**Key configuration points:**
- `assetPrefix`: Ensures assets are served from the correct embedded path
- `rewrites`: Maps the embedded path to root for proper routing
- `images.remotePatterns`: Allows loading images from Contentful
- `webpack`: Configures SVG handling to use @svgr/webpack for React components

### 2. Install SVG Dependencies

```bash
npm install --save-dev @svgr/webpack
```

### 3. Configure TypeScript

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 4. Create Global Type Definitions

Create `global.d.ts` in the root:

```typescript
declare module '*.svg' {
  import { FC, SVGProps } from 'react';
  const content: FC<SVGProps<SVGElement>>;
  export default content;
}

declare module '*.svg?url' {
  const content: string;
  export default content;
}
```

This allows importing SVGs as React components or URLs.

### 5. Configure Tailwind CSS

Update `tailwind.config.ts`:

```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        background: 'var(--background)',
        foreground: 'var(--foreground)',
      },
    },
  },
  plugins: [],
};

export default config;
```

**If using shadcn/ui components**, install and configure:

```bash
npx shadcn@latest init
```

Answer the prompts to set up shadcn/ui with your preferred style. This will create `components.json`.

### 6. Configure PostCSS

Update or create `postcss.config.mjs`:

```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

### 7. Configure ESLint

Update `eslint.config.mjs`:

```javascript
import { dirname } from 'path';
import { fileURLToPath } from 'url';
import { FlatCompat } from '@eslint/eslintrc';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const compat = new FlatCompat({
  baseDirectory: __dirname,
});

const eslintConfig = [
  ...compat.extends('next/core-web-vitals', 'next/typescript'),
  ...compat.extends('prettier'),
  {
    rules: {
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-unused-vars': [
        'warn',
        {
          argsIgnorePattern: '^_',
          varsIgnorePattern: '^_',
        },
      ],
    },
  },
];

export default eslintConfig;
```

Install ESLint dependencies:

```bash
npm install --save-dev eslint-config-prettier eslint-plugin-prettier @eslint/eslintrc
```

### 8. Configure Prettier

Create or update `.prettierrc`:

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false
}
```

Install Prettier:

```bash
npm install --save-dev prettier
```

### 9. Set Up Environment Variables

Create `.env.production`:

```bash
#MSAL
NEXT_PUBLIC_AZURE_AD_CLIENT_ID=75de59c0-2a7f-4d4a-baa6-fcc8987e1615
NEXT_PUBLIC_AZURE_AD_TENANT_ID=6bec8931-53ed-4238-a6a8-495fa78471d5
```

**Note:** These are the standard Haven MSAL credentials. Check with the team if different credentials are needed for your specific app.

Create `.env.example` for documentation:

```bash
#MSAL
NEXT_PUBLIC_AZURE_AD_CLIENT_ID=your-client-id-here
NEXT_PUBLIC_AZURE_AD_TENANT_ID=your-tenant-id-here
```

**Important:** Add `.env` to `.gitignore` (it should already be there).

### 10. Update globals.css

Update `src/app/globals.css` to include Tailwind and any custom styles:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  --background: #ffffff;
  --foreground: #171717;
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: #0a0a0a;
    --foreground: #ededed;
  }
}

body {
  color: var(--foreground);
  background: var(--background);
  font-family: Arial, Helvetica, sans-serif;
}

@layer utilities {
  .text-balance {
    text-wrap: balance;
  }
}
```

### 11. Install Core Dependencies

Install the essential dependencies for the Ribbon app:

```bash
npm install @azure/msal-react @tanstack/react-query nuqs axios dayjs
npm install --save-dev @tanstack/react-query-devtools
```

**Dependency purposes:**
- `@azure/msal-react`: Microsoft Authentication Library for authentication
- `@tanstack/react-query`: Data fetching and caching
- `nuqs`: URL state management
- `axios`: HTTP client for API calls
- `dayjs`: Date/time manipulation

### 12. Install UI Dependencies (if needed)

For shadcn/ui and related UI components:

```bash
npm install class-variance-authority clsx tailwind-merge lucide-react
npm install @radix-ui/react-dialog @radix-ui/react-slot @radix-ui/react-tabs
```

### 13. Install Shared Haven Packages

Identify which Haven packages the original module uses and install them:

```bash
# Example - adjust based on your specific module needs
npm install @havenengineering/experience-admin-styleguide
```

**How to identify needed packages:**
1. Check the source module's `package.json` dependencies
2. Look for `@havenengineering/*` packages
3. Install the ones that contain business logic or shared components

### 14. Add SASS Support (if needed)

If migrating styles gradually from SCSS:

```bash
npm install sass
```

This allows you to import `.scss` files during the migration period.

### 15. Commit Configuration Changes

```bash
git add .
git commit -m "chore: configure build tools and dependencies

- Configure Next.js with embedded path support
- Set up SVG handling with @svgr/webpack
- Configure TypeScript, Tailwind, ESLint, Prettier
- Install core dependencies (MSAL, React Query, etc.)
- Set up environment variables

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

## Verification Checklist

Before moving to Phase 3, verify:

- [ ] `next.config.ts` configured with correct embedded path
- [ ] SVG webpack configuration added
- [ ] TypeScript configuration updated
- [ ] `global.d.ts` created for SVG types
- [ ] Tailwind CSS configured
- [ ] ESLint and Prettier configured
- [ ] `.env.production` created with MSAL credentials
- [ ] Core dependencies installed
- [ ] Haven shared packages installed
- [ ] Project builds without errors: `npm run build`
- [ ] Dev server runs: `npm run dev`
- [ ] Linter runs: `npm run lint`

## Test the Configuration

Run these commands to verify:

```bash
# Install all dependencies
npm install

# Run linter
npm run lint

# Build the project
npm run build

# Start dev server
npm run dev
```

All commands should complete successfully.

## Common Issues

**Issue: SVG imports cause TypeScript errors**
- Ensure `global.d.ts` exists and is in the TypeScript include path
- Verify `@svgr/webpack` is installed

**Issue: Tailwind classes not working**
- Check `tailwind.config.ts` content paths include your source files
- Verify `globals.css` has Tailwind directives
- Restart dev server after config changes

**Issue: ESLint config not loading**
- Ensure `@eslint/eslintrc` is installed
- Check for syntax errors in `eslint.config.mjs`

**Issue: Build fails with module not found**
- Verify all dependencies are installed: `npm install`
- Check for typos in import paths
- Ensure `tsconfig.json` paths are configured correctly

## Next Steps

Once Phase 2 is complete and verified, proceed to:
**[Phase 3: Authentication Setup](./03-authentication-setup.md)**

## Notes for Claude

When executing this phase:
1. Replace `{app-name}` with the actual module name throughout
2. Check the source module's `package.json` to identify all needed Haven packages
3. Don't install packages that aren't needed for this specific module
4. Verify the build succeeds before moving to Phase 3
5. Document any module-specific configuration needs
