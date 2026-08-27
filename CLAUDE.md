# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a React + TypeScript + Vite frontend application for an R&D Operations Management System (ProRnD) for IIT Guwahati. It manages the lifecycle of research projects, funding, staff management, and financial operations, and integrates with several backend services:

- **Frappe/ERPNext** — the primary backend (doctypes, workflows, auth)
- **Appwrite** — internal messaging (conversations, attachments)
- **Ledger API** (external Go service) — project account-head payments/commits
- **MinIO** — file storage
- **Attendance API** — leave/absence data

## Development Commands

```bash
yarn install      # install dependencies
yarn dev           # start dev server on port 8081 (HMR enabled)
yarn build          # production build (outputs to dist/)
yarn lint            # eslint
yarn preview          # serve a production build locally
```

There is no test script configured in this project.

### Production deploy

`deploy-prod.sh` orchestrates a production run under PM2: it runs `vite build --mode production`, and if a sibling `../rndopsapp` directory exists, copies the build into it (`dist/` → `../rndopsapp/public/frontend/`, `dist/index.html` → `../rndopsapp/www/rndopsapp.html`) for serving from within the Frappe app. It then runs `vite preview` on port 8081 (override with `PORT`). If `../rndopsapp` isn't present, it just serves the build directly — this is the path used when the app is deployed standalone behind Nginx (see below) rather than embedded in a Frappe app directory.

### Nginx-fronted dev/preview

The dev server (and `vite preview`) are also reachable through Nginx at `http://pragati.iitg.ac.in` (proxying to `172.16.135.118:8081`), so `vite.config.ts` explicitly allows that host (`server.allowedHosts` / `preview.allowedHosts`) and lets the HMR client infer its port from `window.location` rather than hardcoding one, so HMR keeps working whether you're hitting the dev server directly or through Nginx on port 80.

## Architecture Overview

### Backend Integration (Frappe)

- **FrappeProvider**: wraps the app, connects via `frappe-react-sdk`, `siteName="prornd.local"` ([App.tsx](src/App.tsx))
- **Dev proxy** ([proxyOptions.ts](proxyOptions.ts)): routes `^/(app|api|assets|files|private)` to the Frappe backend (host/port from `VITE_APP_BACKEND_HOST` env var + `webserver_port` from `common_site_config.json`), plus separate proxies for the other services below. `vite preview`'s proxy (`previewProxyOptions`) deliberately excludes `assets` from the proxied prefixes, since the built SPA's own bundle is served from `/assets/*` and would otherwise collide with Frappe's static asset path.

### External Service Integrations

Several features talk to services other than Frappe, each proxied separately in dev (see [proxyOptions.ts](proxyOptions.ts)) and reached directly (or via Nginx) in production. Hosts/ports for all of these are configured via `VITE_*_HOST` / `VITE_*_PORT` env vars (`.env`, `.env.production`, `.env.local`).

- **Messaging (Appwrite)**: [src/lib/appwrite.ts](src/lib/appwrite.ts) configures the Appwrite client (Account/Databases/Storage) against `/appwrite` in dev (proxied, since the Appwrite SDK requires an absolute URL — resolved against `window.location.origin`). Used by [src/services/messagingService.ts](src/services/messagingService.ts), `src/pages/messages/`, and hooks like `useConversations`, `useMessages`, `useUnreadCount`, `useAppwriteSession`.
- **Ledger API**: [src/services/ledgerService.ts](src/services/ledgerService.ts) calls the external Go ledger service via `/ledger-api` (rewritten to `/api` upstream). Used for project account-head payments and commit status (`SETTLED`, `PARTIALLY_PAID`, `OVERPAYMENT`, `COMMITTED`, `PENDING`) in `ProjectLedgerModal`, `ProjectLedgerFull`, `CommitPayment`, etc.
- **MinIO**: file storage proxied via `/prod-rnd-files`.
- **Attendance API**: proxied via `/attendance-api` (rewritten to `/api` upstream); used directly with `fetch()` in `LeaveModule`/`LeaveModuleForm` for absence data.

### State Management

- **SWR** via `frappe-react-sdk` for Frappe data fetching/caching — global config in [App.tsx](src/App.tsx) disables auto-revalidation to avoid unnecessary backend calls
- **Zustand** for global state
- **React Context** for theme (dark/light)

### Routing Architecture

React Router v7, defined in [main.tsx](src/main.tsx), with role-based access control via `AuthRouteWrapper` (each protected route specifies an `allowedRole`). The router's `basename` comes from `VITE_BASE_PATH`.

- **Public routes**: landing page (`/`), `login`, `x-admin`
- **Role-based dashboards**: Director, Dean, Head, HoS R&D, R&D Staff, Project Staff (`Dashboard` handles role-based redirection itself, so it isn't wrapped in `AuthRouteWrapper`)
- **Main workflow areas**: Projects (registration, proposal, endorsement, details/overview, analytics, ledger), Funding (fund sanction, fund received, deposit slips, proforma invoice), Applications (travel, TA/DA settlement, temporary advance, reimbursement, direct purchase, leave), HR (staff recruitment, resignation, honorarium), Messaging, Tasks (pending tasks, task registry, payments)
- **Role types** referenced in `allowedRole`: `All_ProRnd_User`, `Permanent Employee`, `Director`, `Dean, RnD`, `Hos, RnD (Head of Section, RnD)`, `staff, RnD`, `project staff`, `head_approver_1`, `Ado_RnD`

### Custom Hooks

Located in [src/hooks/](src/hooks/):

- **useFrappeClientScript**: emulates Frappe's client-side scripting environment in React — registers event handlers (`onchange`, `refresh`) for Frappe doctypes and provides mock helpers (`flt`, `cint`, `$`) to execute Frappe Python-to-JS scripts, syncing form state with those scripts
- **useFrappeFetchFrom**: handles Frappe's `fetch_from` field linking
- **useProjectBudget**: project budget allocation calculations
- **useDepositSlipCalculations**: deposit slip form calculations
- **useStableFrappe**: stable references to Frappe SDK methods
- **useConversations / useMessages / useUnreadCount / useConversationUnreadCounts / useAppwriteSession**: Appwrite-backed messaging state

### Dynamic Form System

- **DynamicFormRenderer** ([src/components/forms/DynamicFormRenderer.tsx](src/components/forms/DynamicFormRenderer.tsx)): renders forms from Frappe field metadata, supports field dependencies (`depends_on`, `mandatory_depends_on`, `read_only_depends_on`) evaluated via [evalExpression.ts](src/utils/evalExpression.ts); handles Data, Link, Select, Textarea, Date, Check, Attach, Table, etc.
- **ChildTableComponent**: renders/manages child tables (Frappe table fields)
- **API Service Layer** ([src/services/apiService.ts](src/services/apiService.ts)): centralized endpoints per doctype/module (Rate Contract, Travel, TA/DA Settlement, Resignation, Temporary Advance, etc.), each exposing `getFields`, `save`, `submit`, `getWorkflowActions`, `performAction`

### UI Component Architecture

- **Base UI**: [src/components/ui/](src/components/ui/) — Radix UI primitives styled with Tailwind CSS via `class-variance-authority`; includes a neo-brutalism design system, global loader, sidebar
- **Business components**: [src/components/](src/components/) — e.g. WorkflowTimeline, ActivityStream, ProjectLedgerModal, action-button sets per module (Temporary Advance, TA/DA Settlement, Travel), utility components (DepartmentName, BudgetHeadName, ProjectTitle)
- **Layout**: [src/components/RndSidebar.tsx](src/components/RndSidebar.tsx) — role-based menu, pending-task badges, Frappe user image

### Styling

- Tailwind CSS ([tailwind.config.js](tailwind.config.js)), beige/neutral palette (`#F9F7F2`), dark mode support
- Assamese/Hindi/English text support in the header (`.assamese-text`, `.hindi-text`, `.english-text`)
- Lucide React + React Icons

### Workflow System

- Workflow actions fetched dynamically per doctype from the Frappe backend
- **WorkflowTimeline**: visualizes approval stages
- [src/utils/workflowUtils.ts](src/utils/workflowUtils.ts): maps workflow states to UI states, determines available actions from current state + user role

### Authentication & Authorization

- `useFrappeAuth` (frappe-react-sdk)
- `AuthRouteWrapper` enforces route-level role requirements
- `useUserRoles` fetches/caches the current user's Frappe roles

### Data Fetching Patterns (Frappe)

1. Fetch document: `useFrappeGetDoc(doctype, name, options)`
2. Fetch list: `useFrappeGetDocList(doctype, options)`
3. Call server method: `useFrappeGetCall` / `useFrappePostCall`
4. Create/update: `useFrappeCreateDoc`, `useFrappeUpdateDoc`
5. File upload: `useFrappeFileUpload`

### Important Patterns

- **Dates**: Frappe dates are `YYYY-MM-DD`; use `date-fns` for manipulation
- **Numbers**: `flt()`/`cint()` mimic Frappe's float/int coercion; `to-words`/`num-words` for number-to-text
- **PDF generation**: `html2canvas` + `jspdf`
- **Debouncing**: `use-debounce` for search/filter inputs
- **Tables**: `@tanstack/react-table`

### Key Architectural Decisions

1. **No auto-revalidation**: SWR is configured to not revalidate on focus/reconnect, to avoid unnecessary backend calls
2. **Global loader**: shown only on route navigation, not on data revalidation
3. **Command palette**: global search via Cmd+K ([CommandPalette.tsx](src/components/CommandPalette.tsx))
4. **Multi-backend**: beyond Frappe, the app talks directly to Appwrite, an external Ledger API, MinIO, and an Attendance API (see "External Service Integrations" above) — each proxied separately in dev
5. **Multi-language support**: header shows the institution name in Assamese, Hindi, and English

## Common Development Workflows

### Adding a New Form Page

1. Add API endpoints in [src/services/apiService.ts](src/services/apiService.ts)
2. Create a page component in `src/pages/application/`
3. Use `DynamicFormRenderer` with form fields from the Frappe backend
4. Wire up `useFrappeClientScript` if the doctype has client scripts
5. Add a route in [src/main.tsx](src/main.tsx) with the appropriate `AuthRouteWrapper allowedRole`
6. Add a menu item in [src/components/RndSidebar.tsx](src/components/RndSidebar.tsx) if needed

### Working with Frappe Client Scripts

1. Fetch the client script from the backend
2. Call `useFrappeClientScript(script, formData, setFormData, doctype)`
3. The hook executes Frappe's client-side logic (field calculations, validations, etc.)
4. `onchange_<fieldname>` and `refresh` events are registered automatically

### Debugging Backend Integration

- Frappe: check the Network tab for `/api/method/` calls, verify the proxy is routing correctly, confirm the Frappe session cookie is present
- Appwrite: requests go through `/appwrite` in dev; check `VITE_APPWRITE_*` env vars are set and the endpoint resolves against `window.location.origin`
- Ledger/Attendance/MinIO: check the corresponding `/ledger-api`, `/attendance-api`, `/prod-rnd-files` proxy entries in [proxyOptions.ts](proxyOptions.ts) and the matching `VITE_*_HOST`/`VITE_*_PORT` env vars

## Path Aliases

`@/` aliases to `src/` (configured in [vite.config.ts](vite.config.ts) and [tsconfig.json](tsconfig.json)).

Example: `import { Button } from "@/components/ui/button"`

## Environment Configuration

A `common_site_config.json` file is expected at the project root for the Frappe proxy target (`webserver_port`).

Key environment variables (Vite `VITE_*`, in `.env` / `.env.production` / `.env.local`):

- `VITE_BASE_PATH` — router basename
- `VITE_APP_BACKEND_HOST` / `VITE_APP_BACKEND_PORT` / `VITE_APP_BACKEND_REGISTRATION_PORT` — Frappe backend
- `VITE_LEDGER_HOST` / `VITE_LEDGER_PORT` — external Ledger API
- `VITE_MINIO_HOST` / `VITE_MINIO_PORT` / `VITE_MINIO_ALT_PORT` — file storage
- `VITE_APPWRITE_HOST` / `VITE_APPWRITE_PORT` / `VITE_APPWRITE_ENDPOINT` / `VITE_APPWRITE_PROJECT_ID` / `VITE_APPWRITE_PROJECT_NAME` / `VITE_APPWRITE_DATABASE_ID` / `VITE_APPWRITE_CONVERSATIONS_COLLECTION_ID` / `VITE_APPWRITE_MESSAGES_COLLECTION_ID` / `VITE_APPWRITE_ATTACHMENTS_BUCKET_ID` — messaging
- `VITE_ATTENDANCE_HOST` / `VITE_ATTENDANCE_PORT` / `VITE_ATTENDANCE_SSO_PORT` — attendance/leave data
- `VITE_ASSET_HOST` / `VITE_ASSET_PORT`, `VITE_CANDIDATE_API_URL`, `VITE_FRAPPE_URL` — misc.
