# CASSIA — No-Login Re-Engineering & Deploy Stabilization

**Date:** 2026-07-01 (re-engineering) → 2026-07-07 (final FK fix & full verification)
**Version:** v2.12.1 · no-login · anonymous workspace
**Deployed:** Render (Docker, free tier) — `cassia-utwt.onrender.com`
**Repo:** `github.com/sanghyun-s/cassia`

---

## Goal

Remove the login/authentication wall from the deployed app so anyone can open it
and use every feature immediately — while **retaining** the full account system in
the codebase (not deleting it), and keeping all functionality intact.

**Business rationale:** a name/email gate in front of a *free* portfolio tool
suppresses adoption. For a top-of-funnel demo, zero-friction access is the
stronger product decision. The account system stays in the code, dormant and
re-enablable, if the product ever needs teams or per-user sign-in.

---

## What was done

### 1. Backed up before touching anything
- Git tag `pre-nologin-backup` and branch `backup/with-auth` at the pre-change commit.
- Every file edit produced a timestamped `.bak`.
- All work staged on a `nologin` branch, merged to `main` only after local verification.

### 2. Diagnosed the "blank deployed page" (it was never a bug)
The deployed site appeared blank/stuck. Root-cause chain, established from logs:
- The local repo was **3 commits behind** GitHub — it was missing the entire
  Docker/Render deploy setup (`Dockerfile`, `render.yaml`, `.dockerignore`,
  CORS + relative-API-path fixes). Synced local `main` to the deployed commit.
- Two earlier deploy failures were **missing-module** errors (`passlib`, then the
  `uploads` package excluded by `.dockerignore`) — already fixed in later commits.
- The live deploy was actually **healthy**. The "blank page" was the **login wall**
  plus free-tier **cold start** (~30–60s wake). The 401s in the logs were the
  login gate doing its job — not a failure.

**Lesson:** read the runtime logs before theorizing. A "blank page" on a PaaS can
be routing, a crash on import, a missing data build, or — as here — the app working
exactly as written and simply gated behind login.

### 3. Removed login by *neutering*, not deleting (Option B)
Deleting `auth.py` would have broken boot (`main.py` imports `User` and
`get_current_user` from it → `ModuleNotFoundError`). Instead:
- **`auth.py`** — `_resolve_current_user` returns an **anonymous per-browser
  `User`** derived from a cookie token (`anon_<token>`), honoring any legacy real
  session first. Never 401s a normal visitor.
- **`main.py`** — an `@app.middleware("http")` mints/sets the anon cookie on first
  visit; `SIGNUP_INVITE_CODE` made optional.
- **`auth_router.py`** — `/auth/signup`, `/auth/login`, `/auth/logout` return
  **410 Gone**; `/auth/me` returns the anonymous user (200).
- **`index.html`** — login gate removed (app always shows), 401 re-flush made
  inert, and a dismissible **functional-cookie notice** added.

Nothing deleted; the container boots identically; every feature preserved.

### 4. Deployed and updated docs
- Merged `nologin` → `main`; Render auto-deployed. Confirmed live: opens straight
  to chat, "👤 guest · connected", cookie notice shows.
- README updated in two passes to reflect no-login reality **honestly**: intro,
  Quick-start, banner, Security section, architecture diagram, endpoint reference,
  project layout, plus a dated "Login removed for deployment" roadmap milestone —
  while keeping the Phase 5 auth work documented as **built-but-dormant** (accurate
  engineering history, not erased).

### 5. Caught and fixed a real bug before pitching (the important one)
Pre-launch testing revealed `POST /sessions → 500` and, downstream,
`POST /sessions/undefined/uploads/ingest → 404` (no session id because session
creation failed).

**Root cause:** `core_topics` / `core_saves` declare
`user_id TEXT NOT NULL REFERENCES users(user_id)`, and connections run
`PRAGMA foreign_keys=ON`. The anonymous identity (`anon_<token>`) was **never
inserted into the `users` table**, so the first *write* failed the foreign-key
constraint. Reads worked (hence "guest · connected" looked fine), which is why it
was invisible until the first create/upload.

**Fix (mirrors the existing `ensure_default_user` tombstone pattern):**
- Added `ensure_user_exists(user_id)` to `session_store.py` — idempotent
  insert-if-missing into `users` (email `NULL` to dodge the `UNIQUE` constraint).
- `auth.py` calls it when resolving the anon identity (lazy import, non-fatal
  wrapper), so the `users` row exists before any FK-bearing write.

Verified in isolation (reproduced the exact `IntegrityError` without the fix;
confirmed session + core writes succeed with it; confirmed idempotent), then
end-to-end on the running app.

### 6. Full feature verification (post-fix)
Exercised on the running app: `POST /sessions → 200`, PDF upload + RAG ingest
(14 chunks), SQL Q&A (Apple Q1 net income, with visible generated SQL and result
table), chart build, CSV/MD/HTML export, save/delete controls. All green.

---

## Working method (what made this safe)

- **Read-only locators first** → paste real code → write **surgical, reversible
  appliers** (timestamped `.bak`, idempotent sentinels, uniqueness-checked anchors),
  verified with `py_compile` / `node --check` in an isolated container before ever
  running on the machine.
- **All-or-nothing appliers:** if an anchor is missing or ambiguous, the patch
  **aborts with no change** — never half-applies. (This caught a genuine ambiguity:
  a literal `</body>` inside the HTML export template vs. the real closing tag.)
- **Test before advertising.** The FK bug loaded fine and passed on reads; only a
  write exposed it. Finding it in pre-launch testing — not after posting a public
  "try it" link — is the difference between a normal fix and an embarrassment.

---

## Talking points for a pitch / write-up

- **Product judgment over feature-hoarding:** recognized that auth — a correct
  Phase 5 decision for a "deploy-ready product" — worked *against* a free
  acquisition funnel, and made the deliberate call to remove the wall while
  preserving the capability. Positioning decision, not a walk-back.
- **Reversible, disciplined engineering:** every change backed up, branch-isolated,
  compile-verified, and idempotent; honest docs that reflect the shipped reality
  without erasing real work.
- **Debugging maturity:** separated "looks broken" from "is broken" via logs
  (the blank page was a cold-started login wall, not a crash); traced a 500 to a
  foreign-key constraint and fixed it at the root rather than papering over it.
- **Pre-launch QA discipline:** caught a write-path bug that read-path smoke tests
  missed, before it reached users.

---

## Rollback / safety net (still in place)

- Branch `backup/with-auth`, tag `pre-nologin-backup` → the full account version.
- `.bak` files for every patched file (no-login patch, README passes, FK fix).
- The account system remains in the code, dormant — re-enablable without a rewrite.

## Known trade-offs (deployed free tier)

- Sleeps ~15 min idle → first visit after a quiet spell waits ~30–60s (warm the
  URL before a demo).
- Ephemeral disk → anonymous saves reset on redeploy. A persistent disk mounted at
  `/app/outputs` is the future fix for durable saves.
- Anonymous identity is per-browser: clearing cookies / a new browser = a fresh
  workspace (expected).

## Launch status

- Launch announcement posted (repo link; app link held back deliberately).
- App remains deployed and fully functional.
- LinkedIn pitch to be drafted separately.
