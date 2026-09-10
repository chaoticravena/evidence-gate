---
name: evidence-gate
version: 1.0.0
description: Pre-deploy audit across 8 fixed categories (cost, isolation, config, abuse, deploy, error UX, supply chain, documentation), where every "pass" requires concrete evidence — not an LLM's confidence that it read the code and it looked fine.
license: MIT
---

# evidence-gate

Pre-production audit that scans a codebase across 8 fixed categories before anything ships. Built on one premise that most audit skills skip: a "pass" without evidence is worse than no audit, because it creates false confidence. This skill requires proof for every category marked clean — a test run, a real request against a real environment, an actual command output — not a summary of what the code looked like on read-through.

## How this differs from a generic ship-gate/quality-gate skill

Most pre-deploy audit skills report pass/fail/manual per category based on reading the code. That's useful for catching obvious misses, but it silently converts "I read this and didn't see a problem" into "this is verified" — which is a different claim. evidence-gate keeps those two states separate: a category is only "✅ verified" if something was actually run and produced a real result; otherwise it's "reviewed, not tested" and stays visibly unverified, even if nothing suspicious turned up. The audit never quietly upgrades a read-through into a pass.

## Intercept behavior

When the user signals deploy intent ("deploy", "ship it", "push to production", "go live", or the equivalent in their language):

1. Ask: "Have you run evidence-gate? Want me to run it now?"
2. If yes, run the full audit below.
3. If the user says they already ran it, ask when. If more than 24h ago, or code changed since, recommend re-running.
4. Never skip this silently — the point is to interrupt the deploy reflex, not rubber-stamp it.

## Step 0: Detect stack and scope

Before running any category, determine:

- Test runner and command (`make check`, `npm test`, `pytest`, etc.) — don't assume, check `package.json`/`Makefile`/CI config first.
- Env file convention (`.env.example`, `.env.sample`, or none) — if none exists, note that as a category 3 finding itself, don't skip the check.
- Scope for this run: full repo, or diff-only.
  - Full (default for pre-deploy / first run / more than a few days since last run): scan the whole codebase across all 8 categories.
  - Diff-only (`/evidence-gate diff`): scan only files changed since the last commit, tag, or deploy marker (whichever the user specifies, or the last commit if unspecified). Use this for routine mid-sprint checks — it's what makes running the skill weekly practical instead of only at launch time. A diff-only run is never a substitute for a full run immediately before an actual deploy; say so if the user tries to treat it as one.

### Time-boxing

State a time-box per category before starting (a few minutes each is typical — adjust for repo size). If a category can't reach a real evidence-backed conclusion within its box, stop it, mark it "not verified — [reason: ran out of time / needed access not available / scope too large]", and move to the next category. Don't let one category consume the whole run.

## Categories (fixed scope — don't improvise a new category mid-run, and don't silently drop one)

1. **AI/LLM cost** — does every endpoint calling an LLM have a cost ceiling (circuit breaker on spend), not just a rate limit? A rate limit alone doesn't bound cost if each call is expensive.
2. **Cross-tenant data isolation** — does any cached, precomputed, or shared-queue data get served back to a different user/tenant without revalidating ownership? This is the single most common serious bug this category of audit exists to catch — a cache or fallback path that was correct for one user and silently wrong for the next. (See example note below.)
3. **Environment/config** — does every critical env var (secrets, base URLs, API keys) fail loudly at boot if missing, or does it silently fall back to a dev value (localhost, empty string)? Compare `.env.example` against what actually runs in production.
4. **Rate limiting and abuse** — does every endpoint that sends email, calls a paid third-party API, or triggers a costly action have protection? List which do and which don't — don't assume "most do" is sufficient.
5. **Deploy and migrations** — does the deploy process swap containers atomically? Is there a documented rollback procedure if a migration fails mid-way? Is HTTP caching (CDN/proxy) configured so it can't serve stale HTML against fresh assets or vice versa?
6. **Error messages and failure UX** — are user-facing errors specific (not generic "something went wrong")? A generic error hides the root cause from both the user and whoever debugs it later.
7. **Dependencies and supply chain** — was any new dependency added since the last evidence-gate run without review — especially a third-party skill or package running with production access?
8. **Decision documentation** — do changes that affect production behavior (rollback steps, cache config, cost limits) live somewhere that survives the end of the chat session (e.g. `DEPLOY.md`, `DECISIONS.md`), not only in conversation history?

**Example** (illustrative, not a rule to copy literally): a caching layer that stores a computed affiliate link keyed by product ID instead of by (product ID, user ID) will serve user B the link generated for user A. The rule in category 2 is the general pattern — reused shared state without ownership revalidation — not this specific scenario.

## Verification discipline

- A finding is only "✅ verified" with concrete evidence: a test run with shown output, a command executed with real output, a request made against a real environment with a captured response. "Read the code, looks fine" is NOT verification — report it as "reviewed, not tested" instead, never as passed.
- Never report "no findings" in a category without stating what was actually scanned. If a category couldn't be verified (missing tool, no access, scope too large for available time), report it explicitly as "not verified" — don't let the category silently read as clean.
- Any check that touches a real environment (production or a staging environment with real data) must be cleaned up afterward — test accounts, temporary scripts left on a server. Confirm by name what was removed.
- Severity, 3 levels:
  - 🔴 Blocker: breaks real usage now, or exposes data/credentials across users.
  - 🟡 Risk: doesn't break normal use, but has a real abuse/cost scenario.
  - ⚪ Cosmetic: no functional impact.
- Never fix anything during the audit — this is a findings pass, not an editing pass. Report findings ordered by real severity, not by discovery order, and wait for a decision on what becomes a follow-up fix.

## Output format

```
## evidence-gate — [date]

### 🔴 Blocker
[finding]: [file:line] — [description] — Evidence: [how it was confirmed, or "not verified, requires manual test"]

### 🟡 Risk
[same format]

### ⚪ Cosmetic
[same format]

### Not verified
[category]: [why it couldn't be verified]

### Summary
[N] blockers, [N] risks, [N] cosmetic, [N] categories not verified.
Recommendation: [safe to deploy / do not deploy until X is resolved / review category Y before deciding]
```

## Rules

- 8 fixed categories — don't invent a new one mid-audit, and don't skip one without reporting it as "not verified."
- "Passed" requires real evidence, never a code read-through alone. Without evidence, the status is "reviewed, not tested" — never "✅".
- Never fix anything during the audit — only surface and classify findings.
- Clean up any real-environment test artifact before closing, by name.
- Report findings ordered by real severity, not discovery order.

## Options

- `/evidence-gate` — run the full audit now.
- `/evidence-gate [category]` — run a single category. Use only for a quick recheck after a targeted fix; a pre-deploy decision should be based on a full run, not a partial one.
- `/evidence-gate diff` — run all 8 categories, scoped to files changed since the last commit/deploy marker. For routine checks between deploys, not a substitute for a full run right before shipping.
