# Frontend Guidelines for b2b-hunter

This document outlines the frontend setup, design principles, and technologies that power the b2b-hunter web application. It’s written in everyday language so anyone—technical or not—can understand how the user interface is built and maintained.

## 1. Frontend Architecture

### Overview
- **Next.js (React framework)**
  • Handles both server-side rendering (SSR) for fast initial loads and client-side navigation for app-like interactivity.  
  • Pages and routes follow a file-based convention, keeping URLs and code organized.
- **React**
  • Builds UI elements as self-contained components (buttons, forms, cards) that are easy to reuse and update.
- **Tailwind CSS**
  • A utility-first styling approach that applies small, consistent classes directly in markup, avoiding large custom style sheets.

### Scalability & Maintainability
- **Modular Components**: Each UI part lives in its own folder, making it simple to find, update, or swap out functionality.  
- **Folder Structure**:
  • `/pages` for top-level routes (dashboard, search, company detail, settings)  
  • `/components` for reusable UI pieces (atoms, molecules, organisms)  
  • `/layouts` for common page wrappers (sidebar, header, footer)
- **Code Splitting**: Next.js automatically splits JavaScript bundles by page, so users only download what they need.
- **Configuration in Code**: Tailwind’s config file centralizes colors, fonts, and breakpoints, ensuring consistent styling and easy theme updates.

## 2. Design Principles

### Key Principles
1. **Usability**: Forms and flows guide users step by step. Inline validation and clear error messages keep interactions smooth.  
2. **Accessibility**: We follow WCAG 2.1 AA standards—semantic HTML, meaningful alt text, keyboard navigation, and sufficient color contrast.  
3. **Responsiveness**: Mobile-first layout ensures everything adapts from small phones to large monitors.  
4. **Consistency**: Reusable components and utility classes guarantee the same look and feel everywhere.  
5. **Performance**: Fast load times and snappy UI feedback maintain user focus and satisfaction.

### Applying Principles
- All forms use labeled inputs and real-time feedback (e.g., React Hook Form + Yup).  
- Interactive elements (buttons, links) have clear hover and focus states.  
- Charts and data tables include ARIA attributes and live region updates for screen readers.
- Sidebar and header navigation are always visible on desktop and toggleable on mobile.

## 3. Styling and Theming

### Styling Approach
- **Utility-First** with Tailwind CSS—no hand-rolled CSS files except overrides in `styles/globals.css`.  
- **Purge Unused Styles** at build time to keep CSS bundles lean.

### Theming
- Defined in `tailwind.config.js`: primary, secondary, accent, neutrals, and feedback colors.
- Supports light and (optionally) dark modes via a `theme` context and Tailwind’s `dark:` variants.

### Visual Style
- **Modern Flat Design** with subtle shadows and rounded corners.
- **Minimal Glassmorphism** touches (transparent panels with slight blur) only in modals and overlays.

### Color Palette
- **Primary**: #1E40AF (dark blue)  
- **Primary Light**: #3B82F6 (blue)  
- **Secondary**: #10B981 (green)  
- **Accent**: #F59E0B (amber)  
- **Neutral Light**: #F3F4F6 (gray-100), #E5E7EB (gray-200)  
- **Neutral Dark**: #111827 (gray-900)  
- **Success**: #10B981  • **Warning**: #F59E0B  • **Error**: #EF4444  • **Info**: #3B82F6

### Typography
- **Font Family**: Inter, sans-serif fallback.  
- **Headings**: bold, scale from 1.5rem to 2.25rem.  
- **Body**: 1rem for readability, with 1.5 line-height.

## 4. Component Structure

### Organization
/components
  • ui/       — Atoms (Button, Input, Icon)
  • form/     — Form controls and validation wrappers
  • layout/   — Sidebar, Header, Footer
  • search/   — FilterPanel, ResultsGrid, CompanyCard
  • dashboard/— SummaryWidget, TrendChart
  • modal/    — ConfirmDialog, ExportOptions

Each component folder contains:
- `index.tsx` (component code)
- `types.ts` (props definitions)
- `styles.module.css` (optional overrides)
- `*.test.tsx` (unit tests)

### Benefits of Component-Based Design
- **Reusability**: Build once, use everywhere—reduces duplication.  
- **Isolation**: Bugs stay local to a component, making fixes straightforward.  
- **Parallel Development**: Teams can work on different components without conflict.

## 5. State Management

### Client-Side State
- **React Context & useReducer** for global concerns:
  • AuthContext (user info, JWT token)  
  • ThemeContext (light/dark mode)
- **Local State** via `useState` and custom hooks for component-specific data (e.g., form inputs).

### Data Fetching & Caching
- **Next.js Data Functions**:
  • `getServerSideProps` for dashboard metrics and initial search results  
  • `getStaticProps` for infrequently changing pages (help, about)
- **Client-Side Updates**:
  • `fetch` or `axios` calls to our backend API  
  • Custom hooks wrap fetch logic, handle loading and error states

## 6. Routing and Navigation

- **Next.js File-Based Routing**:
  • `/dashboard`                    — Main overview page  
  • `/search`                       — Company search and filter  
  • `/company/[companyId]`         — Company detail and contacts  
  • `/settings`                     — Profile, security, billing  
  • `/admin/users`, `/admin/logs`  — User management and audit (admin only)

- **Navigation Components**:
  • `<Link>` from `next/link` for fast client-side transitions  
  • Active route highlighting in the sidebar  
  • Redirects for protected routes (login guard and role checks)

## 7. Performance Optimization

- **Server-Side Rendering** speeds up first meaningful paint and improves SEO.  
- **Automatic Code Splitting** by page and dynamic imports for large modules (charts, maps).  
- **Tailwind Purge** removes unused CSS classes.  
- **Next/Image** for optimized image loading (lazy loading, responsive srcsets).  
- **Caching**: HTTP caching headers on static assets and SWR or in-memory cache for client data.  
- **Lazy Loading** of non-critical components (modals, admin pages).  
- **Bundle Analysis**: Regular checks with `next-bundle-analyzer` to keep bundles small.

## 8. Testing and Quality Assurance

### Unit & Integration Tests
- **Jest** with **React Testing Library** for component and hook tests.  
- Test user interactions (form validation, button clicks, conditional rendering).  
- Mock API calls to verify loading and error states.

### End-to-End (E2E) Tests
- **Cypress** for critical user flows:
  • Sign-up, email verification, sign-in  
  • Company search and filtering  
  • Profile detail view and export  
  • Settings changes and plan upgrade

### Linting & Formatting
- **ESLint** enforces JavaScript/TypeScript best practices.  
- **Prettier** keeps code style consistent across the team.  
- **Husky** pre-commit hook to run linting and tests before every push.

### Accessibility Testing
- **jest-axe** for automated a11y checks in unit tests.  
- **Manual audits** with browser devtools and screen readers on key pages.

### Continuous Integration
- **GitHub Actions** runs linting, tests, and type checks on every pull request.  
- Code reviews ensure two eyes sign off on UI changes and accessibility.

## 9. Conclusion and Overall Frontend Summary

The b2b-hunter frontend combines Next.js, React, and Tailwind CSS to deliver a fast, responsive, and accessible user experience. We follow clear design principles—usability, accessibility, and performance—and organize our code into reusable, well-tested components. Routing, state management, and styling are centralized in familiar conventions, making it easy for new team members to jump in and contribute. Automated tests, CI pipelines, and linting protect quality, while modern optimizations keep the app snappy. Together, these guidelines ensure that b2b-hunter remains maintainable, scalable, and delightful for our users as it evolves.