# CASSIA — Dev Note: No-Login Deployment & the Persistence-as-Responsibility Journey

**Span:** 2026-06-28 (deploy) → 2026-07-12 (final copy alignment)
**Version:** v2.12.1 · no-login · anonymous workspace
**Deployed:** Render (Docker) — `cassia-utwt.onrender.com`
**Repo:** `github.com/sanghyun-s/cassia`

> A running record of a multi-step decision loop: removing the login wall for a
> public demo, diagnosing what "broke" after deploy, fixing the one real bug, and
> arriving at a deliberate design boundary — *ephemeral demo workspace, export as
> the permanent record, no data custody* — with the account system preserved for a
> possible market-grade future. Written both as a record and as raw material for a
> later public "design retrospective" post.

---

## 0. One-line summary

The hard part was never the code. It was recognizing that **persistence in a
no-login accounting demo is not only an engineering question — it's a
data-responsibility question** — and drawing the boundary accordingly.

---

## 1. Starting decision — freeze auth, don't delete it

CASSIA was built through Phase 5 with full authentication: bcrypt sign-in,
server-side session cookies, per-user data isolation, defense-in-depth in both the
API and the vector store. Correct for a "deploy-ready product."

For a **public portfolio demo**, a name/email gate is friction that suppresses the
one thing a demo needs: people actually trying it. Decision (mutually agreed with
mentor): **remove the login wall, keep every feature.**

Key engineering choice: **neuter, don't delete.** `main.py` imports `User` and
`get_current_user` from `auth.py`; deleting auth would break boot. Instead the
identity resolver returns an **anonymous per-browser `User`** (`anon_<cookie>`),
`/auth/*` endpoints return `410 Gone`, and the login gate is removed from the UI.
The account system stays in the codebase, dormant and re-enablable.

**Takeaway:** "remove a feature" and "delete its code" are different operations.
Neutering kept the app booting identically and preserved the Phase-5 work as an
asset, not a liability.

---

## 2. "The deployed page is blank" — separating *looks broken* from *is broken*

Post-deploy, the site appeared blank/stuck. Investigation (via Render logs, not
assumption) found **no bug**:

- The local repo was **3 commits behind** GitHub — missing the entire Docker/Render
  deploy setup. Synced local `main` to the deployed commit.
- Two earlier deploy failures were **missing-module** errors (`passlib`, then an
  excluded `uploads` package) — already fixed in later commits.
- The live deploy was **healthy**. The "blank page" was the **login wall** plus a
  free-tier **cold start**. The 401s in the logs were the gate doing its job.

**Takeaway:** on a PaaS, "blank page" can be routing, an import crash, a missing
data build, or the app working exactly as written behind a gate. Read the runtime
logs before theorizing; the log names the layer in one line.

---

## 3. The one real bug — anonymous identity had no `users` row (FK failure)

Pre-launch testing (create session → upload) surfaced `POST /sessions → 500`, then
a downstream `.../undefined/uploads/ingest → 404`.

**Root cause:** `core_topics` / `core_saves` declare
`user_id TEXT NOT NULL REFERENCES users(user_id)`, and connections run
`PRAGMA foreign_keys=ON`. The anonymous identity (`anon_<token>`) was **never
inserted into `users`**, so the first *write* failed the foreign-key constraint.
Reads worked (hence "guest · connected" looked fine) — which is why it was invisible
until the first create/upload.

**Fix (mirrors the existing `ensure_default_user` tombstone pattern):** an
idempotent `ensure_user_exists(user_id)` inserts the anon user row (email `NULL` to
dodge the `UNIQUE` constraint) at identity resolution, before any FK-bearing write.
No schema change, no logic rewrite. Verified in isolation (reproduced the exact
`IntegrityError` without the fix; confirmed writes succeed and are idempotent with
it), then end-to-end on the running app.

**Takeaway:** removing auth had a structural side effect two layers away — the anon
identity was incomplete without a backing row. It only surfaced on the first
*write*, which is exactly why read-path smoke tests missed it. **Test the write
path before advertising.**

---

## 4. The deeper turn — persistence is also a data-responsibility question

After launch, a longer-idle test showed saved sessions/Core resetting. Diagnosis:
**not a new bug** — Render's free tier spins down on idle and cold-starts on a fresh
container with an **ephemeral disk**; the anonymous saves live on that disk. Same
ephemeral behavior previously noted "on redeploy," now also triggered by
spin-down. The save/recall *logic* was correct; the *storage substrate* was
transient.

This raised a real product question — is CASSIA a demo, or a tool people return to?
Two options were on the table:

- **Persistent disk / Postgres** — durable storage, real "return to your workspace."
- **Recovery-key identity** — durable profile without a password.

Both were rejected, deliberately, for **this** app:

- **Durable storage makes you a data custodian.** A demo that retains strangers'
  uploads — for an *accounting* tool that might touch SSNs/EINs/client records —
  inherits obligations (where is it stored, how is it deleted, is it encrypted)
  that a portfolio piece shouldn't carry.
- **Recovery keys are login in another form.** A code the user must save and re-enter
  reintroduces the exact friction the auth-freeze removed.

**Decision:** keep the demo **ephemeral-by-design**; **export (MD/HTML/CSV) is the
permanent record.** This is not a limitation to apologize for — it's the correct
architecture for a no-login, no-retention, portfolio-grade tool.

**Takeaway:** the strongest engineering move was *not* adding durability. Choosing
data-minimization on purpose — weighing feature-richness against user friction,
data responsibility, and product purpose — is the senior decision.

---

## 5. Getting the facts right — where does the data actually live?

Rather than assume, this was verified:

- **Local `outputs/` was untouched** by the deploy (last local session 6/9, Chroma
  6/12) — the container never writes back to the developer's machine. Strangers'
  uploads never land locally. Good.
- **Render shell** (`ls -la /app/outputs`) showed `coreckoner.db` present with a
  **live, recent mtime** — proving sessions/saves are written **server-side in the
  container**, not in the browser. The browser holds only the identity cookie.
- The spin-down reset itself is the proof: browser-stored data would survive a
  server restart; these saves didn't — so they lived on the server's ephemeral disk.

This corrected an imprecise earlier framing ("browser-level") to the accurate one:
**held transiently on the demo server during the session; not on the user's device;
not retained.** That precision matters because it properly motivates the
"don't upload sensitive data" boundary — files do touch a server the user doesn't
control.

*(Note: the Render plan was later moved Free → Starter — solely to regain Shell
access, which the free tier disables. Side effect: Starter stays warm, so idle
spin-down no longer resets data; redeploys still do, since no persistent disk was
added. No data-custody path was taken.)*

**Takeaway:** for any claim that doubles as a safety statement, verify against the
running system instead of reasoning from the local repo. Go look at the container.

---

## 6. Aligning the story — docs & UI copy to match reality

Because the pitch/README were written assuming the features as *built* (durable,
per-user), the auth-freeze created a claim/behavior gap. Closed it in copy-only,
reversible passes (no logic, schema, identity, or storage touched):

- Removed production-grade permanence claims ("permanent core," "all future
  sessions," "even months later," "private knowledge base").
- Reframed Core as a **working scratchpad**; **export** as the durable record.
- Corrected "per-browser" → **transient server-side, not on your device, not
  retained**.
- Strengthened the **don't-upload-sensitive-data** guidance by attaching its
  *reason*: files are processed on the demo server, which does not encrypt at rest.
- Preserved the **built-vs-deployed** distinction: Phase-5 auth/architecture stays
  documented as real history; only the *deployment's* durability is hedged.

**Takeaway:** when a deployment decision changes behavior, the docs and the pitch
are part of the deliverable. Honest copy that matches the running system is a
feature, not an afterthought.

---

## 7. Parked for a market-grade future (not a gap — a roadmap)

Auth genuinely made CASSIA better for **simulations and internal demos** — richer,
per-user, persistent. It is the *wrong* choice for a **public portfolio demo**,
where friction and data-custody are liabilities. These aren't contradictory; they're
different contexts.

If CASSIA ever becomes market-grade — real users returning to durable workspaces —
the natural next phase is: **re-enable auth** (the Phase-5 stack is preserved) +
**persistent storage** (disk or external DB) + **encryption at rest** + a real
identity/deletion model. The work is scoped and ready; it's a deliberate
*when-justified* decision, not unfinished business.

---

## 8. The throughline (for the retrospective post)

- Product judgment over feature-hoarding: removed a correct-but-wrong-context
  feature on purpose, and preserved it for when the context changes.
- Debugging maturity: separated "looks broken" from "is broken" via logs; traced a
  500 to a foreign-key constraint and fixed it at the root.
- Pre-launch QA: caught a write-path bug that read-path checks missed.
- The real insight: **"I thought persistence was a feature question; testing the
  deployed app showed it was also a data-responsibility question."**
- Discipline throughout: backups, branch isolation, compile-verified idempotent
  reversible patches, and docs kept honest to the running system.

Tone for any public version: **design retrospective, not apology.** The auth-freeze
and the ephemeral-by-design boundary were correct decisions; the only real drift was
a copy/claim mismatch, which was found and fixed — a normal correction loop.

---

## Rollback / safety net (all intact)

- Branch `backup/with-auth`, tag `pre-nologin-backup` → the full account version.
- `.bak` files for every patch (no-login, FK fix, README/UI passes).
- Account system remains in the code, dormant, re-enablable without a rewrite.

## Commit trail (key points)

- No-login merge → `main` (auth neutered, features intact).
- Anon user-row FK fix (`f9b2583`) — makes writes succeed for anonymous identity.
- Persistence copy alignment (`fc37609`) — remove permanence over-claims.
- Transient server-side copy accuracy (`d0c3816`) — server-side/transient framing +
  strengthened sensitive-data caution.
