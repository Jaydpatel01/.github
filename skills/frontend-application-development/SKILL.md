---
name: frontend-application-development
description: Build production-quality, fast, accessible, and maintainable web interfaces. Use when developing frontend features, UI components, React/Vue/Angular applications, or web application styling and interactions.
---

# Skill: Frontend Application Development

## Purpose
Build a production-quality web interface that is fast, accessible, maintainable, and provides an excellent user experience. The frontend is what users interact with directly — it must be held to the same engineering standards as any other system component.

## When to Use
Apply this skill when building a new frontend application, adding significant features to an existing UI, or performing a frontend architecture overhaul.

## Technology Selection

### Framework
- **React 18+ with TypeScript**: Default choice. Largest ecosystem, excellent tooling, strong typing.
- **Vue 3 with TypeScript**: Excellent DX, gentler learning curve, great for teams new to component frameworks.
- **Next.js**: React with server-side rendering, file-based routing, API routes. Best for SEO-sensitive applications.
- **Nuxt 3**: Vue equivalent of Next.js.
- **SvelteKit**: Minimal bundle size, excellent performance. Best when bundle size is a primary concern.

### Build Tool
- **Vite**: Fast HMR, excellent developer experience. Default for Vue, React, Svelte.
- **Turbopack / Next.js built-in**: For Next.js projects.

### Styling
- **Tailwind CSS**: Utility-first. Excellent for rapid development and consistent design systems.
- **CSS Modules**: Good for component-scoped styles without a framework dependency.
- **Styled Components / Emotion**: CSS-in-JS. Use only when dynamic theming requires it.

### State Management
Choose based on complexity:
- **React built-in (useState + Context)**: Sufficient for most small-to-medium apps.
- **Zustand**: Lightweight, minimal boilerplate, TypeScript-friendly. Preferred for medium-complexity state.
- **TanStack Query (React Query)**: For server state management (data fetching, caching, synchronization). Use this before reaching for Redux.
- **Redux Toolkit**: Only when complex global state with time-travel debugging is genuinely needed.
- **Pinia** (Vue): Lightweight, Composition API-based, TypeScript-first. Default for Vue projects.

### HTTP Client
- **Axios**: Feature-rich, good defaults, request/response interceptors. Use for complex API integrations.
- **TanStack Query + fetch**: Preferred for data-fetching-heavy applications.
- **tRPC**: Type-safe end-to-end API calls when the backend is TypeScript.

## Project Structure

```
src/
├── components/
│   ├── ui/          # Reusable presentational components (Button, Input, Modal)
│   └── features/    # Feature-specific composite components
├── pages/ (or app/ for Next.js 13+)
├── hooks/           # Custom React/Vue hooks
├── stores/          # State management (Zustand stores, Pinia stores)
├── services/
│   └── api/         # API client configuration and typed endpoint functions
├── utils/           # Pure utility functions (no side effects)
├── types/           # Shared TypeScript types and interfaces
├── constants/       # Application constants
├── styles/          # Global styles, Tailwind config, theme variables
└── assets/          # Static assets (images, fonts, icons)
```

## Development Process

### Step 1 — Project Initialization
1. Scaffold with `npm create vite@latest` or `npx create-next-app@latest` — use TypeScript template.
2. Configure ESLint (eslint-plugin-react, eslint-plugin-jsx-a11y for accessibility linting), Prettier, and a pre-commit hook.
3. Set up `tsconfig.json` with strict mode enabled.
4. Configure path aliases (e.g., `@/components` → `src/components`) for clean imports.
5. Add `.env.example` for all required environment variables.

### Step 2 — Design System and Component Architecture
1. Establish a component design hierarchy: tokens → primitives → composites → features → pages.
2. Define design tokens: color palette, typography scale, spacing scale, breakpoints.
3. Build primitive components (Button, Input, Text, Icon) first; feature components consume them.
4. Document components in Storybook if the project has a component library or design system.
5. Enforce single-responsibility: each component does one thing well.

### Step 3 — Routing
1. Implement code-split routing — each route is a lazy-loaded bundle (`React.lazy` / `defineAsyncComponent`).
2. Define route constants as typed enums or const objects; do not hardcode route strings.
3. Implement protected routes that redirect to login when unauthenticated.
4. Implement a 404 Not Found fallback route.
5. Handle deep linking: the app must render correctly from any valid URL, not just the root.

### Step 4 — API Integration
1. Centralize all API calls in a `services/api` module — never call `fetch`/`axios` directly in components.
2. Define TypeScript types for all API request and response shapes.
3. Handle all error states: network error, 4xx client errors, 5xx server errors, timeout.
4. Implement request cancellation for components that unmount before a request completes.
5. Implement loading, success, and error states for every async operation.
6. Use TanStack Query (React Query) for data fetching to get caching, refetching, and background sync for free.

### Step 5 — Authentication Integration
1. Store access tokens in memory (React state / Zustand store) — not `localStorage`.
2. Store refresh tokens in `HttpOnly` cookies (set by the server).
3. Implement an Axios/fetch interceptor to automatically refresh the access token on 401 responses.
4. Implement protected route guards that check authentication before rendering.
5. Implement graceful logout that clears all in-memory tokens and invalidates the server session.
6. Handle auth state across tabs (use `BroadcastChannel` or `localStorage` events for signout).

### Step 6 — Form Handling and Validation
1. Use **React Hook Form** or **Formik** for complex forms; do not reinvent form state management.
2. Define validation schemas with **Zod** or **Yup** — these schemas should be shared with the backend if on the same codebase.
3. Show inline validation errors — only after a field is touched (not on every keystroke).
4. Disable submit buttons while submission is in progress; prevent double submissions.
5. Clear sensitive form fields (passwords) after successful submission.
6. Implement optimistic UI updates for operations where the server response is predictable.

### Step 7 — Accessibility (WCAG 2.1 AA — Non-Negotiable)
1. All interactive elements must be keyboard navigable.
2. All images must have descriptive `alt` text; decorative images use `alt=""`.
3. Color contrast must meet WCAG AA: 4.5:1 for normal text, 3:1 for large text.
4. All form inputs must have visible labels (not just placeholders).
5. Use semantic HTML elements: `<nav>`, `<main>`, `<header>`, `<footer>`, `<article>`, `<section>`.
6. Manage focus correctly after modal open/close and route transitions.
7. Add `aria-live` regions for dynamic content updates (notifications, loading states).
8. Test with a screen reader (VoiceOver on macOS, NVDA on Windows) before shipping.
9. Run `eslint-plugin-jsx-a11y` in CI — fail the build on accessibility linting errors.

### Step 8 — Performance Optimization
**Performance budget (Core Web Vitals targets):**
- LCP (Largest Contentful Paint): < 2.5s
- INP (Interaction to Next Paint): < 200ms
- CLS (Cumulative Layout Shift): < 0.1
- JavaScript bundle (initial load): < 200KB gzipped

**Techniques:**
1. Code splitting: lazy-load routes and heavy components.
2. Image optimization: use WebP/AVIF format, proper `width`/`height` attributes to prevent CLS, `loading="lazy"` for below-the-fold images.
3. Font optimization: `font-display: swap`, preload critical fonts.
4. Memoization: use `React.memo`, `useMemo`, and `useCallback` only where profiling reveals unnecessary re-renders (do not premature-optimize).
5. Virtualization: use `react-virtual` or `@tanstack/virtual` for long lists (> 100 items).
6. Debounce and throttle expensive event handlers (scroll, resize, search input).
7. Bundle analysis: run `vite-bundle-analyzer` or `webpack-bundle-analyzer` to identify large dependencies.
8. CDN: serve all static assets through a CDN; configure long-lived cache headers.

### Step 9 — Error Handling
1. Implement React Error Boundaries to catch and display errors at the component tree level.
2. Show user-friendly error messages — never raw error objects or stack traces.
3. Provide actionable recovery options: "Retry", "Go back", "Return to home".
4. Log frontend errors to an error tracking service (Sentry, Datadog, etc.).
5. Include correlation IDs from API error responses in error reports for cross-system tracing.

### Step 10 — Internationalization (i18n) — if multi-language is required
1. Never hardcode user-facing strings in components.
2. Use **react-i18next** (React) or **vue-i18n** (Vue).
3. Store translations in structured JSON files per locale.
4. Handle pluralization, date formatting, and number formatting correctly per locale.
5. Design layouts to accommodate text expansion (German is ~30% longer than English).

### Step 11 — Progressive Web App (if offline/mobile use is required)
1. Add a Web App Manifest with icons and theme colors.
2. Implement a Service Worker for offline capability and asset caching.
3. Use Workbox for cache strategy management (cache-first for static assets, network-first for API calls).
4. Add an install prompt for "Add to Home Screen".

## Security Checklist
- [ ] No secrets or API keys in frontend code or environment variables that are bundled (these are public)
- [ ] All external URLs validated before use (open redirect prevention)
- [ ] No `dangerouslySetInnerHTML` with user-controlled content (XSS risk)
- [ ] Content Security Policy header configured on the server
- [ ] Subresource Integrity (SRI) for any third-party scripts
- [ ] Dependency audit passing (`npm audit`)

## Testing
- Unit tests with **Vitest** + **Testing Library**: test component behavior, not implementation.
- Integration tests: test user flows across multiple components.
- E2E tests with **Playwright** or **Cypress**: test critical user journeys against a running application.
- Accessibility testing with **jest-axe** in unit tests and Axe DevTools in E2E.
- Visual regression testing with **Chromatic** or **Percy** for design system components (if applicable).

## Deliverables
- Functional, accessible, and responsive UI
- Complete API integration with error and loading states
- Authentication flow with token management
- Form validation and submission handling
- Code-split routing
- Performance budget met (Core Web Vitals)
- WCAG 2.1 AA accessibility compliance
- Component test suite
- Bundle size analysis report
