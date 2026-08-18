# ADR 0001: UI modernization

- **Status:** Proposed
- **Date:** 2026-08-18

## Context

ShaDo's browser client is a small React application in `checklist.ui`. It was
bootstrapped with Create React App and has not evolved with the React tooling
ecosystem:

- `checklist.ui/package.json` pins `react` and `react-dom` to `^16.13.1`,
  `react-scripts` to `3.4.1`, `react-router-dom` to `^5.2.0`, Bootstrap to
  `^4.5.0`, and `react-bootstrap` to `^1.0.1`. The test stack is
  `@testing-library/react` `^9.5.0`, `@testing-library/jest-dom` `^4.2.4`,
  and `@testing-library/user-event` `^7.2.1`.
- `package-lock.json` is lockfile version 2 with 1,677 package entries.
  The resolved `react-scripts` toolchain includes webpack 4.42, webpack-dev-
  server 3.10.3, Jest 24.9, ESLint 6, Babel 7.9, and Workbox 4.3.1. These
  versions are several major generations behind current supported browsers,
  Node runtimes, and React tooling.
- The repository currently has open UI Dependabot pull requests #63, #64,
  #66, #67, #68, #69, and #71 (postcss/react-scripts, ansi-regex,
  ip, es5-ext, follow-redirects, webpack-dev-middleware/react-scripts, and
  express). The number and age of these transitive-security and tooling
  updates demonstrate dependency drift rather than an isolated stale package.
- `build.sh` changes into `checklist.ui`, runs unconstrained `npm install`, and
  copies the generated `build` directory into `checklist.service/public`.
  There is no UI-specific CI workflow, lint script, type-check script, bundle
  budget, or reproducible `npm ci` build gate. The root `package.json` has no
  UI test script; its `build` script delegates to this shell script.
- `src/App.js` mounts a `BrowserRouter` and renders every route in a set of
  sibling `Route` components. `MainOptions` is mounted at `/` without
  `exact`, while the other screens are conditionally shaped by route matching.
  Components use the React Router v5 `props.history`, `props.match`, and
  `component` APIs. There is no route-level error boundary, lazy loading,
  not-found route, or pending route state.
- The component structure is a handful of page-like components
  (`components/create/create.jsx`, `view/view.jsx`, `recent/recent.jsx`,
  `about/about.jsx`, and `header/header.jsx`) with behavior concentrated in
  `View`. `View` owns checklist data, validation, per-item saving flags,
  errors, and polling. It refreshes every three seconds while suppressing
  requests inside a five-second timestamp window (`view.jsx:172-225`), which
  creates duplicated timing logic and can race with edits. State updates use
  object snapshots such as `{ ...checklist, listItems }`, making concurrent
  edits and stale closures difficult to reason about.
- There is no shared state or server-cache layer. Recent checklists are read
  and written directly through `localStorage` in `recent.jsx` and `view.jsx`
  without schema/version validation, storage error handling, cross-tab
  synchronization, or a hook abstraction.
- `src/service.js` is a thin API facade over `middleware/webrequest.js`.
  It hard-codes `/api/checklists`, sets `mode: 'cors'`, always sends a JSON
  content type (including `GET` and `DELETE`), has no timeout, cancellation,
  retry/backoff, request correlation, response schema validation, or
  centralized error normalization. Error paths call `response.json()` even
  when an API response may not be JSON. The UI mutates caught `Error` objects
  and logs failures with `console.log` (`create.jsx:31-40`, `view.jsx`).
- Styling is mostly Bootstrap 4 utility/component classes plus global CSS in
  `index.css` and `App.css`; the latter still contains unused CRA starter
  styles. `header.css` only sizes the logo. There are no design tokens,
  component-level style boundaries, responsive interaction rules, or visual
  regression checks.
- Accessibility is inconsistent. Some controls have labels or ARIA labels,
  but the info icon link in `header.jsx` has no accessible name, destructive
  and icon-only buttons in `view.jsx` lack names/confirmation, loading
  spinners are not announced, errors are not exposed as a live region, and
  auto-refresh can replace content without an announced update. The app also
  relies on placeholder text for several test queries rather than consistently
  exercising accessible names and roles.
- Tests cover only rendering of the logo, create form controls, and main
  options (`App.test.js`, `components/create/__test__/create.test.js`, and
  `components/main-options/__test__/mainOptions.test.js`). There are no tests
  for routing, API success/error behavior, polling, local storage, checklist
  edits, optimistic/concurrent updates, deletion, not-found handling, keyboard
  flows, or accessibility. Tests use older Testing Library APIs and manual
  `unmountComponentAtNode` cleanup.
- `src/index.js` uses the legacy `ReactDOM.render` entry point. CRA's generated
  `serviceWorker.js` is retained but explicitly unregistered. The PWA
  manifest remains boilerplate, so the service-worker code adds maintenance
  surface without providing an offline product contract.

The companion backend modernization ADR is
`docs/adr/0002-backend-modernization.md` (PR #72). The UI target must preserve
the existing `/api/checklists` contract during migration and coordinate
runtime, error, and deployment changes with that ADR.

## Decision

Modernize the UI in place through an incremental migration, keeping the
existing product behavior and API routes while replacing the unsupported
toolchain and implicit data-flow patterns.

### Target stack

1. Move to a maintained, Vite-based React application using the current
   supported React major (React 19 when the migration begins), TypeScript, and
   strict compiler settings. Use Vite for development and production builds,
   with explicit environment variables and a checked-in lockfile.
2. Replace `react-router-dom` v5 with the current React Router major and
   route objects. Define explicit public routes for `/`, `/create`,
   `/view/:checklistId`, and `/about`, plus a not-found route. Add route-level
   code splitting only after the baseline migration is behaviorally stable.
3. Replace Bootstrap 4/react-bootstrap 1 with a deliberately small design
   system. Prefer accessible semantic HTML and CSS modules or colocated
   styles with CSS custom properties for tokens. A component library may be
   introduced only where it supplies tested keyboard and screen-reader
   behavior; it must not recreate the current broad dependency surface.
4. Introduce a typed API client around `fetch` (or a small maintained client)
   with one base URL/configuration module, explicit request/response types,
   abort signals, timeout handling, content-type-aware error parsing, and a
   normalized error model. Keep endpoint functions for checklists and items
   so components do not construct URLs or inspect raw `Response` objects.
5. Use a server-state library such as TanStack Query for checklist fetching,
   mutation lifecycle, cache invalidation, stale data, and refetch intervals.
   Keep local UI state local; use a small context/store only for genuinely
   cross-route concerns such as preferences. Replace the current polling
   workaround with an explicit visibility-aware refetch policy, and leave a
   seam for backend ADR #0002's future push/realtime capability.
6. Adopt current Testing Library packages with user-event and MSW for
   network-level API mocks. Add unit/component tests for route states,
   mutations, polling/refetch behavior, local-storage persistence, and
   normalized errors; add an accessibility test pass with `jest-axe` or an
   equivalent maintained tool. Add a small Playwright smoke suite for create,
   open, edit, complete, delete, and refresh flows.
7. Make developer and release workflows reproducible: `npm ci` in CI and
   `build.sh` (or its replacement) where appropriate, pinned Node/npm
   versions via an existing repository convention, dependency update
   grouping, lint/typecheck/test/build scripts, and CI checks for all of
   them. Preserve the final artifact handoff to `checklist.service/public`
   until deployment is separated.
8. Remove unused CRA starter assets and the unused service-worker path unless
   an explicit offline/PWA requirement is accepted. If offline support is
   desired, reintroduce it deliberately with cache invalidation, update UX,
   and tests rather than retaining generated code.

### Migration principles

- Do not combine a framework/toolchain upgrade with an unreviewed product
  redesign or API breaking change.
- Keep API compatibility at each phase and use adapters at boundaries.
- Prefer functional state updates, explicit loading/error/empty states, and
  accessible names for every interactive control.
- Update dependencies in coherent, tested groups and review lockfile changes;
  do not treat individual transitive Dependabot PRs as the long-term
  dependency strategy.

## Consequences

### Positive

- A maintained build and runtime path reduces exposure to stale webpack,
  Babel, Jest, ESLint, and dependency vulnerabilities and makes Dependabot
  updates actionable.
- Typed, centralized API and server-state handling makes loading, errors,
  cancellation, cache invalidation, and concurrent edits explicit.
- Explicit routes, accessible controls, and realistic network tests improve
  reliability for keyboard, screen-reader, slow-network, and failure cases.
- Vite's faster feedback loop, strict TypeScript checks, and CI gates improve
  onboarding and reduce regressions.
- The phased approach preserves the current backend contract and artifact
  deployment while creating a clean path to backend ADR #0002's future
  platform changes.

### Negative and risks

- The migration temporarily increases complexity because old and new
  component boundaries or JavaScript/TypeScript modules may coexist.
- React Router, Bootstrap, Testing Library, and build-tool upgrades can
  change markup, styling, and test behavior; visual and accessibility review
  is required.
- TanStack Query does not solve multi-writer conflict resolution by itself.
  Until the backend provides versioning or realtime updates, the UI must
  retain conservative refetch and mutation semantics.
- TypeScript conversion and end-to-end infrastructure require sustained
  maintenance, CI time, and team familiarity.
- A Vite build changes environment-variable and static-asset conventions;
  deployment must verify the generated artifact and `/api` behavior behind
  the existing service.

## Alternatives Considered

### Upgrade CRA in place

Rejected as the primary target. It reduces initial code movement but keeps an
unmaintained application model and hides the build configuration behind
`react-scripts`; it also leaves routing, server state, API errors, and test
coverage unresolved.

### Eject CRA and own webpack configuration

Rejected. Ejecting would convert existing tooling debt into repository-owned
configuration and maintenance without improving the application's state,
accessibility, or API architecture.

### Adopt Next.js or another full-stack React framework

Deferred. Server rendering and file-based routing are not required for this
small, API-backed, share-link application, and introducing a second server
runtime would complicate the existing Express artifact deployment. Reassess
if SEO, server rendering, or a platform migration from ADR #0002 creates a
clear need.

### Rewrite the UI in one release

Rejected. A big-bang rewrite would increase regression risk and delay security
and developer-experience improvements. The existing route and API boundaries
support incremental replacement.

### Keep Bootstrap and add only dependency updates

Rejected as insufficient. It may reduce vulnerability noise but does not
address the legacy component API, accessibility gaps, duplicated route
composition, polling races, or absent behavioral tests.

## Migration Plan

1. **Baseline and guardrails.** Record current routes, API payloads, build
   artifact behavior, and critical user flows. Add CI commands for the
   existing UI, lock Node/npm versions, replace `npm install` with
   `npm ci` where compatible, and inventory Dependabot alerts. Do not change
   user-visible behavior.
2. **Toolchain foundation.** Create the Vite entry point and TypeScript
   configuration alongside the existing app, migrate static assets and
   environment handling, and make the new build produce the same
   `checklist.service/public` artifact. Remove CRA-only configuration only
   after CI and local production builds pass.
3. **Routing and shell.** Convert `App`, `Header`, and page boundaries to
   typed route objects. Add explicit not-found and route error states,
   keyboard-accessible navigation, document titles, and responsive layout
   primitives. Retain the existing URLs.
4. **API and server state.** Implement the typed API client and normalized
   errors, then migrate `Create` and `View` to query/mutation hooks. Replace
   direct `localStorage` calls with a versioned persistence hook, make
   refetching visibility-aware, and ensure mutation responses cannot be
   overwritten by stale polling data.
5. **Component and accessibility hardening.** Replace Bootstrap form/input
   composition with semantic, tested components; add labels, live regions,
   named icon buttons, focus behavior, delete confirmation, and explicit
   loading/empty/error states. Remove unused CRA styles and generated PWA
   code unless an offline decision is made.
6. **Test coverage and release confidence.** Update Testing Library and add
   MSW-backed component tests, accessibility checks, and Playwright smoke
   flows. Add lint, typecheck, test, build, and bundle-size or performance
   checks to CI. Test the UI artifact behind the backend deployment path.
7. **Cutover and cleanup.** Switch the root build/deployment path to the
   modern UI, remove compatibility adapters and old dependencies, close or
   supersede stale UI Dependabot PRs with the consolidated upgrade, and
   document the supported local development and release commands. Coordinate
   API/error and runtime changes with `docs/adr/0002-backend-modernization.md`.
