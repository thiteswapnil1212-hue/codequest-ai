# CodeQuest AI

**Learn Python by solving missions, with a mentor that explains — not just grades.**

Live demo (team deployment): https://codequest-xi-eight.vercel.app

CodeQuest AI (original project name: Vortexa) is a guided Python learning workspace. Learners pick a coding mission, write Python in the browser, run it against public tests, and get progressive, rule-based mentor hints when they get stuck. Trusted grading and saved progress are optional and need a server setup.

## What it does

- **Mission catalog:** 23 authored missions in `src/lib/missions.ts`. Each mission has a stable ID/slug, topic, constraints, examples, starter code, and non-solution hint metadata.
- **Coding arena:** a Monaco-based editor at `/arena/[missionId]`, with per-test verdicts, a dashboard, and a missions list at `/missions`.
- **Local Python execution:** Run executes real Python in the browser with Pyodide in a Web Worker — no server or account needed for practice.
- **Trusted Submit:** Submit runs the full public + hidden suite on a configured Judge0 server and only saves a completion when the trusted result passes. Without Judge0 configured, Submit shows an actionable configuration message and never reports false success.
- **Deterministic mentor:** hints are rule-based and evidence-based. No LLM provider is called. Hints explain the mistake category and suggest the next step without revealing the solution.
- **Adaptive learner:** learner patterns and progress are tracked from attempts, with empty states on the dashboard until real data exists.
- **Accounts (optional):** Supabase email sign-in/sign-up with owner-scoped saved progress. Local practice works without it.

Mission drafts under `missions/drafts/` and `Pankaj mission/drafts/` are not automatically promoted into the canonical catalog.

## Run vs. Submit — read this first

| Action | What it checks | Where it runs |
| --- | --- | --- |
| Run (Local Python) | Public tests only | Browser, Pyodide Web Worker |
| Submit (Local Python) | Public tests only | Browser, Pyodide Web Worker |
| Submit (Live Judge0) | Public + hidden tests | Configured Judge0 server, trusted grading |

Local mode is genuine Python execution, but the browser worker is not a server-side security boundary or a trusted grading path. Use Live Judge0 when trusted grading or persistent progress is needed.

## How evaluation works

The execution contract is discriminated by format:

- `function` — invokes the configured Python function with JSON-compatible test arguments and compares its returned value.
- `stdin_stdout` — supplies text through standard input and compares the program's captured standard output exactly.

Public tests are returned by `GET /api/missions/tests` for local browser execution. Private cases stay in the server-only test catalog and are used only for a configured Live Judge0 Submit.

## Tech stack

Next.js 16, React 19, TypeScript, Tailwind CSS, Monaco Editor, Pyodide, Supabase (optional), Judge0 (optional), Vitest. See `package.json` for the full list.

## Quick start — local practice, no account needed

Prerequisites: Node.js 20+ and npm.

```bash
npm install
cp .env.example .env.local
npm run dev
```

Then open the app and verify:

- `http://localhost:3000/missions`
- `http://localhost:3000/dashboard`
- `http://localhost:3000/arena/sum-of-two-numbers`

On Windows PowerShell, use `Copy-Item .env.example .env.local` instead of `cp`. The first local run downloads the version-pinned Pyodide runtime and WebAssembly/standard-library assets from the jsDelivr CDN; initial network access is required unless those assets are already cached.

## Optional setup — accounts and trusted grading

### Supabase project setup

1. Create a Supabase project.
2. Apply all migrations in `supabase/migrations/` in order, including `002_server_owned_progress.sql`. This migration removes browser writes to attempts, completions, and mistake summaries, retaining authenticated owner-scoped reads while reserving writes for the server-only `service_role`.
3. Enable email auth in Supabase Authentication if you want password sign-in and sign-up.
4. Keep the server-side keys in local environment variables only.

```bash
# From the project root, use the Supabase CLI if available:
supabase db push
```

Or paste the SQL from each migration in numeric order into the Supabase SQL editor.

Create your first account through the Supabase Auth UI or the API routes `POST /api/auth/sign-up` and `POST /api/auth/sign-in`. Include the access token as `Bearer <token>` in the Authorization header for protected routes.

### Environment variables

Copy `.env.example` to `.env.local` and replace the placeholders:

- `SUPABASE_URL`, `SUPABASE_ANON_KEY` — project URL and anon key for server-side auth APIs.
- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` — the same project URL and anon/publishable key for browser sign-in. These are public client values; never put a service-role key in a `NEXT_PUBLIC_*` variable.
- `SUPABASE_SERVICE_ROLE_KEY` — service-role key for server-side writes; never expose it via `NEXT_PUBLIC_*`.
- `JUDGE0_API_URL` — required for trusted Submit verification; optional for Local Python Run. Base URL from a Judge0-compatible provider or self-hosted deployment. The app appends `/submissions`.
- `JUDGE0_API_KEY` — optional Judge0 API key. `JUDGE0_AUTH_HEADER`, `JUDGE0_AUTH_PREFIX`, and `JUDGE0_PYTHON_LANGUAGE_ID` (defaults to `71`, Judge0 CE Python 3.8.1) are optional overrides.
- `NEXT_PUBLIC_LOCAL_PYTHON_TIMEOUT_MS` — optional local Pyodide execution timeout in milliseconds (default 10000; clamped between 1000 and 30000).
- `MENTOR_EVIDENCE_SECRET` — optional server-only random secret of at least 32 bytes. Set the same value on all production instances so signed mentor evidence works across requests and restarts. If unset, a process-local random key is used; outstanding evidence expires on restart.

The arena starts in Local Python mode. The local runner supports the catalog's function and stdin/stdout formats with Python standard-library-compatible code. Arbitrary third-party packages, native extensions, and secure hidden-test grading are not provided by this browser runtime.

## Verification checklist

- In Local Python mode, confirm actual Python executes in the browser worker and checks only public cases.
- In Live Judge0 mode with Supabase configured, a persisted attempt is created for an authenticated learner.
- A mission completion row is upserted only when a trusted Live Judge0 Submit result shows all required tests passed.
- Dashboard metrics come from persisted data and show empty states when no data is available.

Run the checks with:

```bash
npm run lint
npm test
npm run build
```

## Security notes

- Learner ownership is enforced from the authenticated user identity. Client-supplied IDs are not trusted for writes.
- Full source snapshots are intentionally not stored; only attempt metadata and classification data are persisted.
- Hidden tests, privileged execution output, and provider internals are never exposed to learner-facing records.
- Run and Submit requests are bounded to 128 KiB total and 64 KiB of source, with an eight-second per-test limit and a 30-second execution deadline. Judge0 is configured with resource caps and network access disabled where the provider supports those fields; the actual isolation guarantees depend on the selected Judge0 deployment.
- The run route uses an in-memory sliding-window limit (5 requests per minute per verified learner; 20 per minute in one shared anonymous bucket). This is process-local and is not globally consistent across serverless instances or restarts. Configure an upstream/shared rate limiter at deployment for production-wide abuse protection.
- No hidden tests or server credentials are sent to Pyodide. Only public test inputs/expected values are supplied by the public-test endpoint.

## Project structure

```text
src/app/            Pages (missions, arena, dashboard, auth) and API routes
src/components/     Arena workspace, editor, mentor panel, mission grid, dashboard
src/lib/            Mission catalog, mentor, adaptive learner, execution, Judge0 client
supabase/           SQL migrations for accounts and saved progress
missions/drafts/    Mission drafts, not yet in the canonical catalog
```

## Team

Built by a team:

- Aman Shejul — team lead
- Swapnil Thite — backend and execution engine (Next.js setup and missions API)
- Pankaj Dandekar — missions and content design
- Yash Ambekar — AI mentor engine

This repository is an imported copy of the original team repository, [AmanShejul/vortexa](https://github.com/AmanShejul/vortexa). Commit history is preserved from the original.
