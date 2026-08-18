# 0001. UI modernization proposal

- Status: Proposed
- Date: 2026-08-18

## Context

The current frontend in `checklist.ui` is a 2020-era Create React App (CRA) single-page application. It is small, functional, and has a clear product scope, but several choices make maintenance and evolution harder than they need to be.

### Current framework and tooling

The app is pinned to a legacy React stack in `checklist.ui/package.json`:

- `react`: `^16.13.1`
- `react-dom`: `^16.13.1`
- `react-scripts`: `3.4.1`
- `react-router-dom`: `^5.2.0`
- `react-bootstrap`: `^1.0.1`
- `bootstrap`: `^4.5.0`
- `@testing-library/react`: `^9.5.0`

This is outdated relative to the current React ecosystem and makes the app vulnerable to stalled tooling, missing ecosystem support, and slower build/dev ergonomics. CRA 3 is especially old; it has effectively been superseded by modern build tooling such as Vite and by React 18+ features, concurrent rendering, and more robust hooks and scheduler behavior.

The generated app still carries default CRA conventions from `checklist.ui/README.md`, plus a custom `serviceWorker` registration in `checklist.ui/src/index.js` and a service worker file at `checklist.ui/src/serviceWorker.js`. The app currently calls `serviceWorker.unregister()`, which means the project is not taking advantage of the PWA tooling and is not reinforcing a modern offline-first architecture.

### Component architecture

The UI is organized as a small set of route-level components rather than a domain-driven feature architecture:

- `src/App.js` defines the top-level router and page layout.
- `src/components/header/header.jsx` renders the top navigation and logo.
- `src/components/main-options/mainOptions.jsx` provides the shared checklist ID form and create route.
- `src/components/create/create.jsx` renders the checklist creation form.
- `src/components/view/view.jsx` holds the single most complex stateful screen: checklist fetch, polling, item editing, save behavior, and delete flows.
- `src/components/recent/recent.jsx` reads browser localStorage and renders cards for recently accessed checklists.
- `src/components/about/about.jsx` is a static informational screen.

This architecture is understandable for a MVP, but the app has already started to centralize behavior in `View` and the route tree is doing a lot of UI orchestration without a shared data/domain layer. There is no separation between view logic, API contracts, and business rules. The main risk is that more checklist behaviors will continue to accumulate in a single component and become harder to test and maintain.

### State management and data flow

The app is stateful but it does not have a formal app state layer. State is primarily managed with React hooks (`useState`, `useEffect`, `useCallback`) in individual components. Examples:

- `src/components/create/create.jsx` maintains `validated` and `title` local form state.
- `src/components/view/view.jsx` stores `checklist`, `validated`, `lastFetch`, `itemSaving`, `showError`, and `errorMessage`.
- `src/components/recent/recent.jsx` reads from `localStorage` directly on render.

The main data flow is direct API calls via a custom helper layer in `src/service.js` and `src/middleware/webrequest.js`:

- `fetchChecklist`, `createChecklist`, `updateChecklist`, `addItem`, `updateItem`, `deleteItem`
- `get`, `post`, `put`, `del`
- a thin `fetch` wrapper with `Content-Type: application/json` and `mode: 'cors'`

This works, but it leaves all loading, concurrency, retry, cache invalidation, optimistic update, and error state handling to imperative component code. The `View` component polls the backend every 3 seconds with `setInterval` and suppresses refreshes if `freshChecklist.lastUpdated <= checklist.lastUpdated`. This is a real-time collaboration design, but it is brittle: stale closures, duplication of fetch logic, and concurrency bugs are all possible when state changes while the timer fires.

### Styling and design system

The UI mixes Bootstrap 4 with custom CSS:

- `src/App.css` defines a small set of app-level layout styles.
- `src/components/header/header.css` only contains a tiny `.logo` rule.
- `public/index.html` loads Bootstrap via CDN with a pinned version (`bootstrap.min.css` at version 4.5.0).
- Components use `react-bootstrap` primitives (`Navbar`, `Form`, `Button`, `Card`, `InputGroup`, etc.).

This is pragmatic for a lightweight app, but it is not a scalable design system. There is no component library tokenization, no theme layer, no dark mode strategy, and no strong separation between structural CSS and product styles. The style stack is also bound to a legacy Bootstrap API and CSS class conventions that are not ideal for modern design and accessibility work.

### Routing and navigation

Routing is implemented with `react-router-dom` v5 (`BrowserRouter`, `Route`) in `src/App.js`:

- `Route path="/" component={MainOptions}`
- `Route exact path="/" component={RecentChecklists}`
- `Route path="/create" component={Create}`
- `Route path="/view/:checklistId" component={View}`
- `Route exact path="/about" component={About}`

This is a workable router setup, but it is not robust for a growing app: the root route is not wrapped in `Switch`, route ordering is implicit, and the same component is mounted on every view in a way that makes route matching more brittle over time. The app also relies on imperative navigation via `props.history.push(...)` rather than a route-aware pattern with typed params and centralized navigation.

### API calls and data contracts

The UI exists as a client against a backend API rooted at `/api/checklists` in `src/service.js`:

- `fetchChecklist(checklistId)`
- `createChecklist(checklist)`
- `updateChecklist(checklist)`
- `addItem(checklistId, item)`
- `updateItem(checklistId, item)`
- `deleteItem(checklistId, item)`

Requests are generic `fetch` calls using the custom middleware in `src/middleware/webrequest.js`. There is no typed API schema, no request validation, no timeout handling, no retry policy, no centralized error boundary, and no consistent data-shape validation. The current code often does `throw new Error(await response.json())`, which eagerly serializes backend errors and is fragile for non-JSON responses. The app has no hydration or caching strategy for checklist data, and the UI is designed around raw backend updates rather than explicit contract-layer modeling.

### Accessibility and UX quality

There are some positive accessibility touches, but the overall implementation is uneven:

- Some controls use `aria-label` (`Checklist checkbox`, `Checklist item value`) and `Form.Control` fields.
- The app includes `Form` validation with `noValidate` and `validated` state in `create.jsx` and `view.jsx`.
- The logo has an alt tag in `src/components/about/about.jsx` and `src/components/header/header.jsx`.

However, the app still relies heavily on placeholders as labels, and several interactive controls are more implicit than semantic. For example, the form in `mainOptions.jsx` uses a placeholder text (`Checklist ID`) as the user-visible label, and the app is not audited for keyboard focus management, focus traps, announcements, or screen-reader-friendly error states. There are no a11y tests and no design review around motion, contrast, or touch targets. The UX is functional but not designed to be a strong, accessible, modern web app.

### Testing and developer experience

The project includes a small set of React Testing Library tests:

- `src/App.test.js`
- `src/components/create/__test__/create.test.js`
- `src/components/main-options/__test__/mainOptions.test.js`

These tests validate basic rendering, but they cover only the happy path and they do not exercise API flows, route transitions, or component behavior under async data updates. There are no integration tests for the checklist editing flow in `view.jsx`, no end-to-end tests, and no CI gating beyond whatever local `npm test` provides. More importantly, the app uses only the default CRA testing setup (`@testing-library/react` 9.5.0 and jest-dom) rather than a modern, richer test stack.

The developer experience is also limited: the project has no TypeScript, no lint rules beyond the default CRA `react-app` config, no shared design tokens, no component API conventions, and no feature-specific tooling. It is appropriate for a one-screen MVP but difficult to scale without a more opinionated frontend architecture.

## Decision

We will modernize the UI around a modern React + TypeScript + Vite SPA, while preserving the product domain and backend contracts. The target stack is:

- React 18.x (stable LTS-era single-page app baseline)
- TypeScript for stronger domain contracts and safer API usage
- Vite for faster builds, dev server, and dependency handling
- `react-router-dom` v6 for route hierarchy and typed navigation
- `@tanstack/react-query` for remote checklist state, caching, and stale-while-revalidate updates
- `react-hook-form` + `zod` for accessible form validation and data validation
- CSS Modules or a small design-system layer, optionally layered on Bootstrap-compatible styles during migration
- Vitest + Testing Library + Playwright for unit, component, and end-to-end validation

This direction fits the actual product shape: the app is a medium-size collaborative checklist UI with a single main workflow and a small number of screens, not a server-rendered application or a complex dashboard. A Vite-based SPA with route-level pages and typed client/server data boundaries is the best balance between modern DX, maintainability, and minimal migration risk.

The modernization will preserve the current product behavior and the API semantics in `src/service.js`, but it will move request handling, validation, and state synchronization to a structured client layer rather than letting every screen directly manipulate fetch results and localStorage.

## Consequences

### Positive

- Better build/dev performance and more predictable local workflow with Vite than CRA 3.
- More reliable UI state via query caching and explicit data fetching semantics.
- Stronger application contracts through TypeScript and validation schemas.
- Better route management and clearer page boundaries with React Router v6.
- Improved component reuse and maintainability with a stronger component model and standardized styling.
- Better test coverage and a lower chance of regressions in checklist editing and collaborative polling flows.
- Easier future work on accessibility, design consistency, and feature additions.

### Negative

- The migration will require deliberate refactoring of the existing `View` logic and localStorage patterns.
- Some older component patterns (`props.history.push`, direct `localStorage`, polling in effects) will need to be rewritten.
- The project must invest in a proper design-system or CSS module strategy; otherwise, the app may remain visually inconsistent.
- There will be a temporary increase in migration effort while legacy code and modernized code coexist.

### Risks

- Polling-based collaboration logic may be hidden in a way that is hard to replace cleanly without product behavior regression.
- Real-time shared-checklist updates could require backend coordination or a more durable state sync model than the current local polling pattern.
- If the migration is done too broadly in one pass, the team could burn time rewriting working code without delivering user value.

## Alternatives considered

### 1. Keep the CRA app and only patch it incrementally

This would require minimal upfront disruption, but it would preserve the legacy toolchain, stale dependency constraints, and brittle state patterns. It would also delay the technical debt and reduce the team’s ability to ship reliable features quickly. This is only suitable as a short-term stopgap, not a long-term strategy.

### 2. Rewrite the frontend in Next.js

Next.js is a strong option for content-heavy apps and apps that need SSR, static generation, or strong SEO. However, this product is a small collaborative checklist SPA, not a large marketing site or content platform. Next.js would add more framework and routing complexity than needed for the current domain.

### 3. Keep the current Bootstrap-only approach and modernize only the CSS

This treats the style system as the main issue, but it does not solve the bigger problem: the app still depends on a legacy React/CRA toolchain and imperative state flow. A design refresh without architecture modernization would leave the underlying risk unchanged.

### 4. Rewrite in another frontend framework (Vue/Svelte)

This could work technically, but it would introduce a larger migration cost without a strong product-level need. The front-end is small and the React ecosystem is already established in the repo. Modernizing within the existing React stack is the lower-risk choice.

## Phased migration plan

### Phase 0: Baseline and freeze the product behavior

- Capture the current user flows and success criteria for checklist creation, viewing, editing, deletion, and recent-list tracking.
- Review the current backend contract and expected behavior from `src/service.js`.
- Add baseline tests around the most important checklist flows before any refactor.

### Phase 1: Replace the build environment

- Create a new Vite-based React 18 TypeScript app under the same app path or a new frontend workspace.
- Move the existing route structure into the new Vite project while preserving URLs and component page names.
- Remove the legacy CRA scaffolding and service worker setup unless it is still explicitly needed.

### Phase 2: Introduce typed domain and API layers

- Define checklist and checklist item models in TypeScript.
- Replace raw `fetch` wrapper calls with a structured client module and shared request/response validation.
- Create a small domain service for create/load/update/delete operations.

### Phase 3: Modernize state management and route model

- Migrate `View` and related flows to React Query for async state, cache invalidation, and error handling.
- Replace direct `localStorage` usage with explicit, testable recent-history logic.
- Move route logic to `react-router-dom` v6 patterns with typed route params and clearer navigation.

### Phase 4: Improve UX and accessibility

- Replace ad hoc form patterns with accessible, typed form components and validation.
- Audit keyboard interaction, focus-order, labels, contrast, and error announcement patterns.
- Convert the mix of Bootstrap classes and custom CSS into a consistent design-system layer or CSS module strategy.

### Phase 5: Harden testing and release quality

- Add component tests around checklist CRUD operations and route behavior.
- Add end-to-end tests for shared checklist flows and error states.
- Add CI coverage gates and a small quality policy for frontend reviews and accessibility checks.

## Summary

The UI is a small but functional React app that has served its purpose, yet it is built on a legacy toolchain and a brittle state architecture. The app’s current design is not unsafe, but it is carrying a mix of 2020-era conventions that will slow development and increase regression risk as the checklist product grows. Modernizing to a Vite + React 18 + TypeScript architecture with typed API contracts, query-based state management, and a stronger testing/accessibility layer is the right technical move for the product’s next lifecycle.
