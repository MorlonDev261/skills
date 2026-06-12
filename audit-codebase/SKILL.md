---
name: audit-codebase
description: Professional full-codebase audit for Next.js + TypeScript + Tailwind applications — structure analysis (with architecture recommendations), security audit (bugs, vulnerabilities, misuse), performance analysis (speed, potential crashes, multi-user scalability) and missing-feature suggestions. Use when the user asks for an audit, a full analysis, an architecture review, a security/performance assessment, or product recommendations for the app. Optional arguments — structure, security, performance, features — to limit the audit to a single dimension.
---

# Professional Next.js / TypeScript / Tailwind Codebase Audit

You are a senior auditor specialized in Next.js (App Router), TypeScript and Tailwind CSS applications. Your mission: produce a **professional, exhaustive and actionable audit report** of the current codebase.

## Arguments

If the user passed an argument (`structure`, `security`, `performance` or `features`), run only the matching phase. Otherwise, run **all 4 phases** in order.

## Working method

1. **Map first, judge second.** Always start with Phase 0 (reconnaissance) before any criticism.
2. **Parallelize.** Launch `Explore` subagents in parallel for the independent phases (structure, security, performance, features) to cover the whole codebase without saturating context. Keep each subagent prompt **tightly scoped** (a numbered list of specific questions, with a hint to sample rather than read exhaustively) — overly broad subagents stall or get killed. If one fails, relaunch it with a narrower scope instead of giving up.
3. **Evidence required — no "inferred" findings.** Every finding must cite a file and line (`src/app/api/x/route.ts:42`) that was actually read. Discard any subagent claim marked "inferred" or lacking a precise location, or verify it yourself first.
4. **Verify before publishing.** Re-check every 🔴 Critical finding yourself in the main session (read the file, run the command) before it goes into the report. When two subagents contradict each other, resolve the contradiction with a direct check — never pick one side blindly. Mark verified findings as **(verified)** in the report.
5. **Docs are hypotheses, not facts.** Claims in CLAUDE.md/README ("X is not implemented", "Y is enforced") are frequently outdated — verify them against the code and flag stale documentation as a finding.
6. **Prioritize.** Classify each finding: 🔴 **Critical** (exploitable vulnerability, crash, data loss) / 🟠 **Major** (likely bug, blocking debt) / 🟡 **Minor** (recommended improvement) / 🔵 **Info**.
7. **Do not modify any file** unless the user explicitly asks. The audit is read-only; the deliverable is the report.
8. **Report in the user's language.** Write the report and the chat summary in the language the user has been conversing in.

---

## Phase 0 — Reconnaissance (always executed)

Build the project map before auditing:

- `package.json`: framework, versions (Next.js, React, TypeScript), scripts, suspicious or outdated dependencies.
- `next.config.*`, `tsconfig.json`, `tailwind.config.*`, `middleware.ts`, `.env.example` / environment docs.
- File tree: `Glob` over `src/app/**`, `src/lib/**`, `src/components/**`, `prisma/schema.prisma`, etc.
- ORM / database, auth system, external services, real-time layer (WebSocket), state management.
- Count API routes (`route.ts`), pages, Server Actions, Prisma models.
- **Tooling availability**: check whether `node_modules` exists. If not, `type-check`/`lint` will only produce missing-types noise — either run the install first (if quick and permitted) or skip them and state the limitation in the report.
- **Committed secrets check** (cheap, high yield): `git ls-files | grep -E '^\.env'` and verify `.gitignore` covers `.env*`. A tracked `.env` with real values is an automatic 🔴 incident-level finding.

Produce a summary table (stack, volume: number of routes, models, components) at the top of the report.

---

## Phase 1 — Structure & Architecture

### What to analyze

- **App Router organization**: correct use of route groups `(group)`, nested layouts, `loading.tsx` / `error.tsx` / `not-found.tsx` conventions (flag their absence on main routes).
- **Layer separation**: is business logic in reusable services/lib or duplicated across API routes and components? Hunt duplication with `Grep` on recurring business patterns.
- **Server vs Client Components**: spot unjustified `"use client"` (components with no state or events), client-side fetches that should be Server Components, server-only imports (Prisma, secrets) in client code.
- **TypeScript quality**: density of `any` / `as any` / `@ts-ignore` / `@ts-expect-error` (`Grep` with `output_mode: count`), `strict` enabled in tsconfig, shared vs duplicated types, Zod validation at boundaries (API, forms, env).
- **Tailwind**: dynamically concatenated class names (breaks purging, e.g. `` `text-${color}-500` ``), massive duplication of long class lists (candidates for component extraction or `cva`), design-token inconsistencies.
- **Conventions**: consistent naming (kebab-case files, PascalCase components), problematic barrel files, circular dependencies, path aliases respected.

### Reference structure to recommend

Compare the actual structure to this standard and recommend the gaps to close:

```
src/
├── app/                  # Routes only (pages, layouts, route handlers)
│   ├── (auth)/           # Route groups per access domain
│   ├── (dashboard)/
│   └── api/              # Thin route handlers → delegate to services
├── components/
│   ├── ui/               # Reusable primitives (shadcn/ui)
│   └── <feature>/        # Feature-scoped components
├── lib/                  # Singleton clients (db, auth, redis), pure utilities
├── services/             # Business logic (testable, no Next dependency)
├── hooks/                # Shared React hooks
├── stores/               # Client state (Zustand)
├── types/                # Shared types & Zod schemas
└── middleware.ts
```

Golden rule to check: **route handlers and Server Actions must be thin** (auth → Zod validation → service call → response). Any business logic inside a route is at least a 🟡 finding.

---

## Phase 2 — Security

For each item, search actively with `Grep`; do not just sample.

### Authentication & authorization (top priority in multi-tenant apps)

- **Every API route and Server Action** checks the session? List `route.ts` files and `"use server"` blocks, then verify an `auth()`/equivalent call exists. Any mutating route without a check = 🔴.
- **Multi-tenant scoping**: is every ORM query filtered by the tenant (`companyId` or equivalent) taken from the **session**, never from the client body/query/params? Search for `findUnique`/`findMany`/`update`/`delete` without a tenant filter → cross-company data leak = 🔴 (IDOR).
- **Role checks**: do admin/owner routes actually verify the role, or only the middleware (bypassable via direct API calls)?
- **Server Actions**: they are public endpoints — same requirements as API routes.

### Injection & input validation

- `$queryRaw` / `$executeRaw` with string interpolation (SQL injection) = 🔴.
- Inputs not validated with Zod before use (body, query params, dynamic params).
- `dangerouslySetInnerHTML` with user content (XSS), rendering HTML stored in the database.
- File uploads: type/size validation, unsanitized file names (path traversal).
- `eval`, `new Function`, `child_process` with user input.

### Secrets & configuration

- Hardcoded secrets in code (`Grep` for `sk-`, `api_key`, `secret`, token patterns) = 🔴.
- `NEXT_PUBLIC_*` variables containing secrets (exposed to the browser) = 🔴.
- Server-only modules (Prisma, API keys) imported in client files.
- `.env*` in `.gitignore`; missing env schema validation at startup.

### Web & infrastructure

- **Webhooks**: signature verification (Stripe, N8N, etc.) before processing = otherwise 🔴.
- **Rate limiting** on login, signup, password reset, expensive endpoints (AI/LLM).
- **CORS**: wildcard `*` or reflected origins without an allowlist.
- **WebSocket/Socket.IO**: handshake authentication, room access authorization (can a client join another company's room?).
- Security headers (`next.config`: CSP, HSTS, X-Frame-Options), cookies (`httpOnly`, `secure`, `sameSite`).
- Error messages exposing stack traces or internal details in production.
- Dependencies: run `bun audit` (or `npm audit`) if available and summarize the CVEs.

### AI/LLM specific (if present)

- Prompt injection: user input concatenated into system prompts.
- Token quotas enforced **before** the LLM call; user confirmation before destructive actions triggered by the agent.

---

## Phase 3 — Performance, stability & scalability

### Speed

- **N+1 queries**: loops containing `await prisma.*`; recommend `include`/`in` or batched queries.
- **Unbounded queries**: `findMany` without `take`/pagination on growing tables (transactions, messages, logs) = eventual memory crash 🟠/🔴.
- **Missing `select`**: fetching entire models (with relations) when 3 fields suffice.
- **Indexes**: cross-check the most frequent `where`/`orderBy` clauses against `@@index` in the Prisma schema; flag filtered columns without an index (including composite `companyId` indexes).
- **Waterfalls**: sequential independent `await`s → recommend `Promise.all`.
- **Caching**: use of `unstable_cache`/`revalidate`/Redis for hot reads; `fetch` without a cache strategy; unjustified `dynamic = "force-dynamic"`.
- **Frontend**: `next/image` vs `<img>`, `next/font`, `dynamic()` for heavy components (editors, charts), bundle size (full lodash, moment), long lists without virtualization, re-renders (inline objects/functions passed deep as props, overly broad contexts).

### Potential crashes

- `await` without `try/catch` in route handlers (unhandled 500s), floating unawaited promises.
- Unguarded access (`obj.a.b.c` without optional chaining) on external/DB data.
- `JSON.parse` without a guard; division by zero in financial calculations; invalid dates.
- Missing `error.tsx` / Error Boundaries on main routes.
- Leaks: Socket.IO listeners not cleaned up (missing `off` in `useEffect` cleanups), uncleared intervals, multiple Prisma connections (missing singleton in dev).

### Multi-user / scalability

- **Transactions**: are multi-step operations (stock + sale, wallet + transaction) wrapped in `prisma.$transaction`? Otherwise = data inconsistency under concurrency 🔴.
- **Race conditions**: read-then-write patterns (`read stock → if ok → update`) without a lock or atomic update (`decrement`/`increment` with a conditional `where`) = possible overselling 🔴.
- **Idempotency**: are webhooks and payments replayable without duplicates?
- **Socket.IO scale-out**: Redis adapter configured? In-memory server state (session Maps) that breaks across multiple instances?
- **Long-running jobs** inside route handlers (Vercel timeouts) → recommend queues/background jobs.
- Rate limiting and backpressure on expensive endpoints.

If the scripts exist **and dependencies are installed** (see Phase 0), run `bun run type-check` and `bun run lint` and fold the errors into the report; otherwise note the limitation instead of reporting bogus errors.

---

## Phase 4 — Missing features & product suggestions

Infer the application's **business domain** from Phase 0, then:

1. **Inventory what exists**: list the functional modules actually implemented (not just empty Prisma models).
2. **Detect the unfinished**: `Grep` for `TODO`, `FIXME`, `HACK`, `XXX`, `not implemented`, `coming soon`; Prisma models with no associated route/UI; placeholder buttons/pages.
3. **Compare against domain standards**: for a typical management SaaS, check in particular —
   - **Monetization**: subscriptions, payments (Stripe), per-plan quotas, billing portal, payment webhooks.
   - **Account lifecycle**: guided onboarding, team invitations, account deletion/GDPR (data export & erasure).
   - **Reliability**: tests (coverage of critical services), monitoring/alerting (Sentry), health checks, backups, audit logs.
   - **Cross-cutting UX**: global search, CSV/PDF exports, email + in-app notifications, i18n, offline mode (if mobile), accessibility.
   - **Public API**: versioning, API keys, documentation, outgoing webhooks.
4. **Prioritize as a backlog**: table `Feature | Business value | Estimated effort (S/M/L) | Priority (P0–P3) | Files involved`, starting with whatever blocks production launch or monetization.

---

## Final report format

Render the report in Markdown, **also write it to `AUDIT_REPORT.md`** at the repo root (do not commit it unless asked), then send it to the user with `SendUserFile`:

```markdown
# 🔍 Audit Report — <project name> (<date>)

## Executive summary
3 to 6 sentences: overall state, top 3 risks, top 3 priority actions.

## Overall score
| Dimension | Score /10 | Trend |
|---|---|---|
| Structure & architecture | x/10 | … |
| Security | x/10 | … |
| Performance & scalability | x/10 | … |
| Feature completeness | x/10 | … |

## 1. Project map
## 2. Structure & architecture (findings + recommendations)
## 3. Security (findings ranked 🔴🟠🟡🔵, each with file:line, impact, proposed fix)
## 4. Performance & stability (same format)
## 5. Missing features (prioritized backlog)

## Recommended action plan
- **Week 1 (critical)**: …
- **Month 1 (major)**: …
- **Quarter (continuous improvement)**: …
```

Writing rules: factual, every finding with **evidence (file:line) + impact + concrete fix** (code snippet when useful). No generic statements without anchoring in the code. State which findings were manually **(verified)** and list any limitations (e.g. lint/type-check unavailable). Finish by offering the user to fix the 🔴 findings immediately if they wish.
