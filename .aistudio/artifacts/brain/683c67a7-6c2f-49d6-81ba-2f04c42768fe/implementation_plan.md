# AutoWallet — Frontend Optimization, Modular Architecture & Full Backend/WebSocket Integration

Optimize and modularize the AutoWallet frontend into a clean, maintainable, and responsive desktop and mobile experience, implement missing pages (`404 Not Found`, `/maintenance` status view, and interactive feature previews), and connect all backend REST, SSE, and API Gateway WebSocket (`/ws`) endpoints while keeping all 68 QA tests green.

---

## User Review & Critical Decisions

> [!IMPORTANT]
> The following architectural and product decisions have been confirmed based on your answers and repository audit:

- **Confirmed Decision 1 — Hybrid Real-Time Transport (`/ws` WebSocket + `/api/analytics/live` SSE)**: The frontend will support both the API Gateway WebSocket (`/ws`) and the FastAPI Server-Sent Events stream (`/api/analytics/live`) with automatic fallback and unified payment/notification synchronization, plus Vite and Nginx proxy forwarding for `/ws`.
- **Confirmed Decision 2 — Complete Backend Endpoint Wiring**:
  - Wire **`POST /api/auth/2fa/disable`** in Security Settings so users can disable two-factor authentication using their current password and either a 6-digit TOTP code or a 32-character recovery code.
  - Wire **`POST /api/ai/chat`** as an automatic fallback when the streaming SSE endpoint (`POST /api/ai/chat/stream`) is interrupted or unsupported by an intermediary proxy.
  - Connect **`GET /health`** and **`GET /api/health`** health checks to power the `/maintenance` status screen and manual service health verification without interfering with existing unit test `fetch` assertions.
  - Preserve explicit read-only notices alongside interactive preview modes for features whose backend write contracts are not yet persisted (password updates, custom rules CRUD, profile edits).
- **Confirmed Decision 3 — Missing Pages & Interactive Previews**:
  - Build a dedicated **404 Not Found** page (`NotFoundPage`) with context-aware navigation (returning to Dashboard when signed in or Login when signed out) instead of silently redirecting unknown URLs to `/login`.
  - Build a dedicated **System Maintenance & Service Status** page at `/maintenance` (`MaintenancePage`) displaying live reachability for the Backend API, API Gateway, and Real-Time stream with one-click retry.
  - Add **Interactive Sandbox Previews** for custom rule simulation in Rules and local profile display preview in Settings while clearly indicating what is persisted on the server vs. previewed locally.
- **Confirmed Decision 4 — Codebase Modularization & QA Test Preservation**:
  - Decompose dense, single-line JSX files (such as `DashboardPage`, `SettingsView`, `SecuritySettings`, and `RulesView`) into focused, readable subcomponents and clean CSS modules while preserving component contracts and all 68 existing QA tests.
  - Configure the workspace root so the AI Studio preview server launches the frontend seamlessly on port `3000` alongside existing Docker/Nginx configurations.

---

## 1. Overview & Core Concept

- **What It Does**: AutoWallet is an automated income-splitting financial workspace (in `MAD`) that routes incoming client payments through prioritized envelope rules (**Rent**, **Tax**, **Savings**, **Free to spend**, and **Main** remainder), backed by live analytics, an AI financial assistant, two-factor security, and GDPR data controls.
- **Target Audience / Persona**: Freelancers, independent consultants, and small business owners managing irregular client payments who need immediate clarity on how much money is locked for bills and taxes versus safe to spend—on both desktop monitors and mobile phones.
- **Key Value**: Eliminates manual spreadsheet budgeting with deterministic rule-based allocations, instant real-time payment alerts via WebSockets and SSE, interactive split previews, and accessible security and privacy controls.

---

## 2. User Experience & Visual Design

- **Key User Flows**:
  1. **Authentication & 2FA Lifecycle**: Users sign in with email/password or Google OAuth, complete 6-digit TOTP or 32-character hexadecimal recovery-code verification, verify email tokens, and manage 2FA activation, recovery code regeneration, and **2FA deactivation** (`POST /api/auth/2fa/disable`) in Security Settings.
  2. **Five-Step Onboarding & Live Split Preview**: New users progress through About → Account → Envelopes → Rules → Try a Payment, seeing real-time allocation math across envelopes before recording their first transaction.
  3. **Responsive Workspace & Real-Time Updates**: Desktop users navigate via a collapsible sidebar (`240px` expanded, `72px` collapsed) and a 3-zone top bar; mobile users (`375px–430px`) use a compact sticky header, ergonomic bottom tab bar with a dismissible "More" sheet, and a thumb-zone floating action button. Incoming payments over `/ws` or `/api/analytics/live` trigger toast alerts, badge counters, and ledger refreshes.
  4. **Rules & Custom Rule Simulator**: Users edit starter rules (`PATCH /api/rules/onboarding`) and can open the interactive **Custom Rule Simulator** to test adding, reordering, or pausing rules against any payment amount before backend persistence is enabled.
  5. **Maintenance & Not Found Recovery**: Visiting `/maintenance` inspects live health endpoints (`/health`, `/api/health`) and guides recovery; visiting an unknown route renders a clear 404 page with direct recovery links.

- **Visual Identity & Theme**:
  - *Aesthetic Direction*: High-craft financial clarity inspired by modern fintech ledgers—calm, structured surfaces, crisp hairline dividers (`1px solid`), single-elevation cards, and zero visual clutter.
  - *Color Palette & Mood*:
    - **60% Neutral Canvas**: Cool Alabaster (`#F3F6FA`) in light mode; Deep Slate (`#0F172A` / `#111827`) in dark mode.
    - **30% Structural Surfaces & Typography**: Crisp white (`#FFFFFF`) or elevated dark slate (`#1E293B`) cards with subtle borders (`#DDE3EA`), deep ink headings (`#1A2330`), and muted slate metadata (`#5E6B7E`).
    - **10% Semantic & Envelope Accents**: Action Indigo (`#5A64B4`), Rent Blue (`#2A78D6`), Tax Terracotta (`#EB6834`), Savings Emerald (`#1BAF7A`), Free Amber (`#EDA100`), and Main Warm Gray (`#C3C2B7`).
  - *Typography & Hierarchy*: `IBM Plex Sans` for UI hierarchy and prose paired with `IBM Plex Mono` and `font-variant-numeric: tabular-nums` across all currency figures, percentages, recovery codes, and timestamps so decimal points align vertically.
  - *Component Styling & Layout*: Single-elevation cards with consistent padding hierarchy, responsive CSS grid workspaces (`1440px` desktop baseline scaling cleanly down to `320px–430px` mobile viewports), and minimum `44×44px` touch hitboxes on mobile controls.

- **Interactive Feedback & Motion**:
  - Fast micro-interactions (`150ms` cubic-bezier transitions on `transform` and `opacity`), skeleton placeholders matching loaded geometry to prevent layout shift, keyboard-accessible dialogs (`Escape` dismissal, focus trapping), and `prefers-reduced-motion` support.

---

## 3. Key Product Decisions & Trade-Offs

- **Decision 1: Dual Real-Time Transport (`/ws` WebSocket + `/api/analytics/live` SSE)**
  - *Chosen Approach*: Implement a unified real-time client that connects to the API Gateway WebSocket (`/ws`) when available while maintaining `/api/analytics/live` SSE streaming with automatic fallback and deduplicated refresh queuing.
  - *Why*: The repository includes both an Express/WS API Gateway (`/ws`) and a FastAPI Redis PubSub SSE stream (`/api/analytics/live`). Supporting both with automatic fallback guarantees live payment notifications regardless of whether the deployment routes through Caddy, Nginx, or direct Vite proxy.
  - *Alternatives Considered*: Replacing SSE entirely with WebSockets would break live payment updates in environments where only the FastAPI backend is running; keeping only SSE would leave the API Gateway WebSocket unused.

- **Decision 2: Connecting `POST /api/auth/2fa/disable` & `POST /api/ai/chat` Fallback**
  - *Chosen Approach*: Add an interactive "Turn off two-factor login" confirmation flow in Security Settings (accepting current password + either a 6-digit TOTP code or a recovery code) and add `askAssistant` (`POST /api/ai/chat`) as an automatic fallback in the Assistant client if SSE streaming fails mid-flight or returns a non-stream response.
  - *Why*: Both endpoints are fully implemented in the FastAPI backend (`backend/app/api/auth.py` and `backend/app/ai/router.py`), and wiring them completes the frontend-to-backend contract.
  - *Alternatives Considered*: Leaving 2FA disable stubbed out would strand users who enable 2FA and later want to turn it off.

- **Decision 3: Modularizing Dense Components Without Breaking QA Tests**
  - *Chosen Approach*: Extract logical subcomponents (Dashboard Sidebar, Header, Mobile Navigation, Link Bank Account Prompt, Security 2FA Disable Form, Custom Rule Preview Drawer, Maintenance Status View, and 404 Page) while keeping existing public component exports, ARIA labels, roles, and initial `fetch` call sequences intact.
  - *Why*: The QA test suite (`68 tests` across `12 test files`) asserts exact `fetch` calls, ARIA roles, and headings. Preserving these contracts ensures 100% test pass rate while making the codebase clean and easy to maintain.

---

## 4. Technical Architecture & Data Strategy *(Technical Reference)*

### Architecture & Component Diagram

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Browser Router & App Shell                            │
│  ┌────────────────┐  ┌──────────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │   Auth Pages   │  │ Onboarding Flow  │  │ Legal / 404  │  │Maintenance │  │
│  │ Login / Signup │  │ Steps 1–5 + Split│  │ Terms/Privacy│  │Status Check│  │
│  │ 2FA / OAuth CB │  │ Preview Engine   │  │ NotFoundPage │  │/health API │  │
│  └───────┬────────┘  └────────┬─────────┘  └──────────────┘  └─────┬──────┘  │
│          │                    │                                    │         │
│          ▼                    ▼                                    │         │
│  ┌──────────────────────────────────────────────────────────────┐  │         │
│  │                    Dashboard Workspace Shell                 │  │         │
│  │  ┌──────────────┐  ┌───────────────────┐  ┌───────────────┐  │  │         │
│  │  │Sidebar / Nav │  │  3-Zone Top Bar   │  │Mobile Tab Bar │  │  │         │
│  │  └──────────────┘  └───────────────────┘  └───────────────┘  │  │         │
│  │  ┌──────────┐ ┌───────────┐ ┌─────────┐ ┌────────┐ ┌───────┐ │  │         │
│  │  │ HomeView │ │Envelopes &│ │RulesView│ │Activity│ │Sandbox│ │  │         │
│  │  │          │ │Detail View│ │+Simulatr│ │& Detail│ │ View  │ │  │         │
│  │  └──────────┘ └───────────┘ └─────────┘ └────────┘ └───────┘ │  │         │
│  │  ┌────────────────┐ ┌─────────────────┐ ┌──────────────────┐ │  │         │
│  │  │ AnalyticsView  │ │  AssistantView  │ │   SettingsView   │ │  │         │
│  │  │ Summary/Charts │ │ Stream + Chat   │ │Profile/2FA/GDPR  │ │  │         │
│  │  └────────┬───────┘ └────────┬────────┘ └────────┬─────────┘ │  │         │
│  └───────────┼──────────────────┼───────────────────┼───────────┘  │         │
└──────────────┼──────────────────┼───────────────────┼──────────────┼─────────┘
               │                  │                   │              │
               ▼                  ▼                   ▼              ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Typed API & Real-Time Client Layer                        │
│  ┌───────────────────┐  ┌───────────────────────┐  ┌──────────────────────┐  │
│  │   REST Clients    │  │  AI Assistant Client  │  │ Live Updates Manager │  │
│  │ Auth / Wallets /  │  │ POST /api/ai/chat/    │  │ WS /ws (API Gateway) │  │
│  │ Rules / GDPR /    │  │ stream + /api/ai/chat │  │ + SSE /api/analytics │  │
│  │ Transactions      │  │ automatic fallback    │  │ /live with backoff   │  │
│  └─────────┬─────────┘  └───────────┬───────────┘  └──────────┬───────────┘  │
└────────────┼────────────────────────┼─────────────────────────┼──────────────┘
             │                        │                         │
             ▼                        ▼                         ▼
┌───────────────────────────────────────────┐     ┌────────────────────────────┐
│       FastAPI Backend (:8000 / /api)      │     │  API Gateway (:3000 / /ws) │
│  Auth, 2FA Enable/Disable, OAuth, Wallets │     │  GET /api/health           │
│  Transactions, Rules, Analytics, AI, GDPR │     │  POST /api/authorize       │
│  GET /health, GET /api/analytics/live     │     │  WS  /ws (JWT Auth)        │
└───────────────────────────────────────────┘     └────────────────────────────┘
```

### Complete Backend & WebSocket Endpoint Mapping

| Endpoint | Method / Protocol | Frontend Module & Handler |
| :--- | :--- | :--- |
| `/api/auth/register` | `POST` | Onboarding Step 5 & Auth signup flow |
| `/api/auth/login` | `POST` | Login card (password, TOTP, or 32-char recovery code) & Security verification |
| `/api/auth/me` | `GET` | Dashboard & Settings user profile loader |
| `/api/auth/link-bank-account` | `POST` | Dashboard unlinked-account banner & Onboarding recovery |
| `/api/auth/oauth/google/login` | `GET` | Browser redirect for Google OAuth sign-in |
| `/api/auth/oauth/google/callback` & `/exchange` | `GET` / `POST` | `GoogleCallbackPage` one-time code exchange |
| `/api/auth/email/verify` & `/email/resend` | `POST` | `VerifyEmailCard` token verification & cooldown resend |
| `/api/auth/2fa/setup` & `/2fa/verify` | `POST` | `SecuritySettings` QR/secret provisioning & recovery codes |
| `/api/auth/2fa/recovery-codes/regenerate` | `POST` | `RecoveryCodes` replacement modal (password + TOTP) |
| `/api/auth/2fa/disable` | `POST` | **Newly Wired** in `SecuritySettings` (password + TOTP or recovery code) |
| `/api/wallets/` | `GET` | Dashboard & Envelopes balance loader |
| `/api/transactions/` | `GET` / `POST` | Activity list, Envelope history & `RecordPaymentDialog` |
| `/api/rules/` & `/api/rules/onboarding` | `GET` / `PATCH` | `RulesView` starter rule editor & Onboarding Step 4 |
| `/api/analytics/summary`, `/timeseries`, `/export` | `GET` | `AnalyticsView` KPIs, charts, and CSV/PDF report downloads |
| `/api/analytics/live` | `GET` (SSE) | `paymentLiveApi` real-time payment stream |
| `/ws` | `WebSocket` | **Newly Wired** WebSocket client for API Gateway real-time events |
| `/api/ai/chat/stream` & `/api/ai/chat` | `POST` (SSE / JSON) | `assistantApi` streaming answer with **Newly Wired** JSON fallback |
| `/api/gdpr/export`, `/delete-request`, `/delete-confirm` | `GET` / `POST` | `PrivacySettings` JSON/ZIP data export & 2-step account deletion |
| `/health` & `/api/health` | `GET` | **Newly Wired** `MaintenancePage` & on-demand service diagnostics |

### Interactive Component & State Mapping

- **2FA Disable Flow (`SecuritySettings`)**: Clicking `"Turn off two-factor login"` opens an inline confirmation form requiring the user's current password and a toggle between a 6-digit authenticator code (`totp_code`) or a 32-character recovery code (`recovery_code`). Submitting calls `POST /api/auth/2fa/disable`, clears in-memory recovery codes, updates `twoFactorEnabled` to `false`, and displays a confirmation banner.
- **Hybrid Real-Time Updates (`paymentLiveApi` + `usePaymentLiveUpdates`)**: Establishes the SSE stream to `/api/analytics/live` and optionally attaches a WebSocket listener to `/ws` when enabled, normalizing incoming payment/ack events into a debounced `200ms` dashboard refresh queue while respecting page visibility, offline events, and `401` session expiration.
- **Custom Rule Simulator (`RulesView`)**: Clicking `"Simulate custom rule"` opens an interactive drawer where users can test adding custom fixed or percentage rules, toggling active/paused states, and adjusting priorities to preview how any payment amount would split across envelopes, while clearly stating that starter rules save to the server and custom rule persistence is a preview.
- **Maintenance & 404 Views (`MaintenancePage` & `NotFoundPage`)**: `/maintenance` renders live health cards for the FastAPI backend and API Gateway with a `"Check status again"` button and a direct link back to `/dashboard` when healthy. Unknown routes render `NotFoundPage` with contextual navigation instead of losing the user's session state.
