# Maison Obsidian Comprehensive Code Review

> Review framework by **Adam Burgess** - https://github.com/qant-au

You are performing a deep, comprehensive code review of **Maison Obsidian**, a single-repository storefront for a batch-atelier fragrance house. The repository root is the project. It contains a Vite + React 19 + TypeScript single-page application (`src/`), Vercel serverless functions (`api/`), a Supabase backend defined as ordered SQL migrations plus Edge Functions (`supabase/`), build and test scripts (`scripts/`), and static assets (`public/`).

You are running as Claude Opus. Token budget is not a concern. Accuracy and completeness are the only constraints. Do not summarise or truncate. Report everything you find.

All paths in this document are relative to the repository root. Run every command from there.

---

## Output Instructions

Save your complete review output as **two separate files**:

1. `reviews/code-review-YYYY-MM-DD.md` - the full review (see Output Format below)
2. `reviews/action-items-YYYY-MM-DD.md` - the distilled action items list (see Action Items Format below). This file becomes **the single working action-items document** for the project - see "Carry-forward protocol" below.

replacing YYYY-MM-DD with today's date (use the `date +%Y-%m-%d` command to confirm).

Save the review file first, then derive the action items file from it. Both files must be written in full in a single operation - do not write incrementally.

### Directory layout - `reviews/` holds only the current cycle

`reviews/` contains exactly three things between cycles: this instructions file, and the **latest** `code-review-YYYY-MM-DD.md` + `action-items-YYYY-MM-DD.md` pair. **Every superseded results file lives in `reviews/history/`.** Write today's two files into `reviews/` (not into `history/`), and archive the previous pair at the end of the cycle - see "Archive the superseded results files" below.

BEFORE beginning the review, check whether any prior results files exist - look in **both** directories:

```
ls reviews/code-review-2*.md reviews/history/code-review-*.md 2>/dev/null
```

If prior reviews exist, read each one. When a current finding was already identified in a prior review, mark it as:

Status: Recurring (previously identified - code-review-YYYY-MM-DD.md)

New findings not previously identified should be marked:

Status: New

This allows trends to be tracked across reviews over time.

### Strikethrough convention in action-items files

Between review cycles the team works through the distilled `action-items-*.md` file and marks progress in place. Understand this convention before a re-review so you can treat already-actioned items correctly:

- When an action item is **completed** or **ruled out**, its `### N.` heading is struck through with `~~...~~` and a one-line resolution note is appended directly under it - `_Done YYYY-MM-DD - <evidence: file:line / commit>._` or `_Won't do YYYY-MM-DD - <reason>._`.
- A **struck** heading is the agreed done-marker. An **unstruck** heading is still open. **Partially**-done items stay unstruck with a `_Partial YYYY-MM-DD - <what's done / what remains>._` progress note.
- This marking happens **only** in the `action-items-*.md` files - **never** in the `code-review-*.md` result files. The review files are the immutable record of the full original findings and must be left intact (they remain the authoritative history a future review diffs against).
- **Take the commit hash AFTER the commit is final** - `git log -1 --format=%h`, once the work is committed - never from a message drafted beforehand. A note written before the commit cites a hash that an amend, a rebase or a conflict resolution then invalidates, and the note is the evidence, so a wrong hash makes the closure unverifiable.
- **Not every hex-shaped token is a git hash.** Deployment IDs, Stripe object IDs and CI run IDs look similar. Read the surrounding prose before calling a hash missing.
- **A hash in an ARCHIVED file is never corrected in place.** Nothing under `reviews/history/` is edited after it lands. Record any correction in the file's `> **CLOSED**` blockquote at the point of archiving.

**Consequence for your review:** for any item already struck in the action-items files, perform a *post-implementation* check - verify the fix actually holds in the current code rather than re-reporting it as new. Focus new findings on unstruck/partial items and on genuinely new issues. If a struck item's fix has regressed or was never really applied, surface that explicitly (Status: Recurring) with the evidence.

### Carry-forward protocol - exactly ONE open action-items list at any time

**Rule: after this review completes, `action-items-YYYY-MM-DD.md` (today's file) must be the only action-items file anywhere under `reviews/` - including `reviews/history/` - containing an unstruck `### N.` heading.** Every older file must be fully struck. The team works one list; a finding that is still open must live in the newest file, not be scattered across several.

Execute this in four steps:

1. **Enumerate every open item across all prior action-items files** before writing anything. Prior files live in `reviews/history/`, plus the outgoing pair still sitting in `reviews/` - cover both:

   ```
   cd reviews && for f in action-items-*.md history/action-items-*.md; do
     [ -e "$f" ] || continue;
     tot=$(grep -c '^### ' "$f"); struck=$(grep '^### ' "$f" | grep -c '~~');
     echo "$f total=$tot struck=$struck open=$((tot-struck))";
   done
   ```

   then list the open headings per file with `grep -n '^### ' <file> | grep -v '~~'`.

   > **Count `~~` anywhere on the heading line, not a `^### ~~` prefix.** Two strike styles occur in
   > practice - `### ~~N. Title~~` and `### N. ~~Title~~` - and a prefix-anchored match silently treats
   > every heading of the second kind as still open. Before concluding a file still has open items,
   > confirm with the `grep -v '~~'` listing above.

2. **Re-verify each open item against the code at HEAD - do not trust the note.** For every open item, determine which of these applies, and record the evidence (file:line / commit):
   - **Still open** - carry it into today's action-items file as its own item, preserving its Why/What/Where, and append to its Refs line: `(carried forward from action-items-YYYY-MM-DD.md #N)`. Update the Where/severity if the code has moved on since it was filed.
   - **Since fixed** (someone closed it without striking it, or the code changed underneath) - do **not** carry it. Strike it in its original file with `_Done YYYY-MM-DD - <evidence>. Verified during the YYYY-MM-DD review._`
   - **No longer applicable** (feature removed, premise false) - do **not** carry it. Strike it in its original file with `_Won't do YYYY-MM-DD - <reason>._`

3. **Close out every prior file.** For each item carried forward, strike its `### N.` heading in the original file with `~~...~~` and append:

   ```
   _Carried forward YYYY-MM-DD - still open, now [action-items-YYYY-MM-DD.md #M](./action-items-YYYY-MM-DD.md). <one line on current state / anything the re-check changed>._
   ```

   Then add a `> **CLOSED YYYY-MM-DD.**` blockquote directly under the file's priority-key line naming today's file as the successor.

4. **Archive the superseded results files.** See the section below - this is the last thing the cycle does.

**These strikes go only in `action-items-*.md` files. Never modify any `code-review-*.md` file** - those are the immutable record.

Today's action-items file opens with a **Carried forward** summary: a short table of every item brought over (`source file #N -> new #M`, title, why it is still open), so a reader can see at a glance what survived from prior cycles versus what is new this cycle.

On the **first** review cycle there are no prior files: say so in the review header, omit Section 9 content beyond a one-line statement, and omit the Carried forward table from the action-items file.

### Archive the superseded results files - the LAST step of the cycle

Once **both** of today's files are written to `reviews/` and the prior files are closed out per step 3, move the **previous** cycle's results pair into `reviews/history/`. Do this last, so `reviews/` shows only the instructions plus the current review at rest.

```
cd reviews && mkdir -p history
TODAY=$(date +%Y-%m-%d)
for f in action-items-*.md code-review-2*.md; do
  [ -e "$f" ] || continue
  case "$f" in *"$TODAY"*) continue ;; esac
  git mv "$f" history/"$f"
done
```

Then verify - this is part of the deliverable, not optional:

```
ls reviews          # exactly: code-review-$TODAY.md, action-items-$TODAY.md, code-review-instructions.md, history/
```

Rules:

- **Only `code-review-*.md` and `action-items-*.md` move.** `code-review-instructions.md` stays in `reviews/` permanently - it is the live prompt, not a result.
- **Use `git mv`**, so history is preserved and the move is one reviewable change. Archiving is a *relocation*, never a deletion - nothing in `history/` is ever edited, deleted or rewritten after it lands there.
- **Fix the path references the move breaks.** Any document outside `reviews/` that links a now-archived file by path must be repointed to `reviews/history/...` in the same commit. Find them with:

  ```
  grep -rn "reviews/action-items-20\|reviews/code-review-20" --include="*.md" --include="*.ts" --include="*.mjs" . | grep -v node_modules | grep -v "^./reviews/"
  ```

  Leave the archived files themselves alone - they are the immutable record.

---

## Project Map

Reconcile this map against the repository at the start of every review (see Review Methodology). It is a snapshot taken 2026-10-07 and will drift; where the code disagrees with it, the code wins and this section should be updated in the same change.

| Area | Path | What it is | Review depth |
|---|---|---|---|
| Serverless API | `api/` (+ `api/_lib/`) | Vercel functions: Stripe checkout, subscriptions, portal, webhook; shipping quote; Claude-backed chat, scent AI and AI fragrance conception; marketing / waitlist | **Deepest** - this is where money, secrets and the LLM meet the internet |
| Database | `supabase/migrations/` | Ordered SQL: schema, Row-Level Security, `SECURITY DEFINER` RPCs, triggers, storage buckets, seed data | **Deepest** - RLS and RPCs are the entire authorisation model for the browser |
| Edge Functions | `supabase/functions/` | Supabase Edge Functions (service-role) | **Deepest** |
| Frontend | `src/` | React SPA: storefront, bag and checkout, auth, Scent DNA experience, customer account, admin console, staff desk | High |
| Build & test scripts | `scripts/` | Function-count gate, output-schema gate, prerender, checkout security test, catalogue import, image tooling | High - these are the project's only automated gates |
| Config | `package.json`, `vite.config.ts`, `vercel.json`, `tsconfig*.json`, `eslint.config.js`, `index.html`, `.gitignore`, `.env.example` | Build, routing, headers, type-checking, linting | High |
| Docs | `README.md`, `docs/`, `STRIPE_INTEGRATION_TODO.md` | Setup, architecture, rollout notes | Medium |
| Static assets | `public/` | Bottle photography and icons | Low - size, naming and licensing only |

### Roles and trust boundaries

Derive the real set from the code each cycle (`supabase/migrations/`, `src/lib/auth.ts`, `src/lib/admin.ts`, `src/lib/staffDesk.ts`, and every `api/` handler). As of 2026-10-07 (reconciled against the code in the first review; the earlier draft of this list repeated the README's stale passphrase and Mailgun descriptions) the code describes:

- **Anonymous visitor / guest shopper** - the browser holds the Supabase anon key; RLS and RPC grants are all that constrain it. Guest orders exist (`0018_guest_orders.sql`) and are matched back to an account by the JWT email claim.
- **Signed-in customer** - Supabase Auth (email/password and Google OAuth); owns their commits, orders, subscriptions, profile and Scentprint.
- **Admin** - membership of an admins table checked by `is_admin()`; catalogue CRUD, inventory, fulfillment, AI conception, marketing, reviews, subscriptions, waitlist.
- **Staff desk** - an **individual Supabase account** that is a member of `staff_members` (or an admin), per `0029_checkout_security.sql`. The shared passphrase introduced in `0025_staff_desk.sql` is gone: `staff_ok()` ignores its `p_pass` argument, `admin_set_staff_passphrase` is revoked, and the README's description of it is stale. Sees order contact details and addresses through the `staff_*` RPCs only.
- **Server** - Vercel functions and the `create-shipment` Edge Function holding `SUPABASE_SERVICE_ROLE_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `ANTHROPIC_API_KEY` (or `ANTHROPIC_KEY`), `AUSPOST_PAC_KEY` and the **Resend** credentials (`RESEND_API_KEY`, `RECOVERY_EMAIL_FROM`); Mailgun appears in the docs but nothing in the code reads it. Anything running with the service role bypasses RLS entirely.
- **Payments** are taken in full at checkout through hosted Stripe Checkout (`api/stripe/checkout.ts`, `mode: "payment"`, no manual capture). The README's authorise-now-capture-later batch model and `capture-batch` function do not exist at HEAD.

### Settled positions - do not file

Record here any position the project owner has decided is deliberate, with the date and the reason, so later cycles do not re-derive it. **None are recorded yet.** Adding to this list is a normal outcome of a cycle: when a finding is closed as "won't do - deliberate", record the position here in the same change.

A settled position whose premise is an operational **state** (pre-launch, paused, test mode) rather than a design decision must carry the date and the condition that ends it, or it will go stale silently.

---

## Review Methodology

Use all tools at your disposal:
- File reads - read key files in full, do not skim
- grep / ripgrep for pattern searches across the codebase
- Directory listings to understand project structure
- Shell commands for dependency checks, git history scans, builds and the project's own test scripts

**Before starting - reconcile the Project Map against disk (mandatory, every review).**

```
git ls-files | sed 's|/[^/]*$||' | sort -u
ls api api/_lib supabase/functions supabase/migrations scripts docs
```

Any directory or top-level area present on disk but absent from the Project Map must be folded into this review at an appropriate depth, AND added to the map in `reviews/code-review-instructions.md` in the same change. A map entry with no corresponding code should be removed. Also run `git status --short` and note anything untracked that looks like it should be tracked or ignored.

**Then check what the documentation claims exists actually exists.** The README and `docs/` describe routes, Edge Functions, migrations and components by name. Diff those names against `git ls-files`. A documented component that is not in the tree (or an undocumented one that is) is a Section 5 finding, and a documented *security* control that is not in the tree is a Section 2 finding.

Work through the project in this order, deepest first:
1. Repository hygiene (Section 1)
2. `supabase/migrations/` in full and in order - schema, every RLS policy, every function (note `SECURITY DEFINER`, `search_path`, and `grant execute ... to anon/authenticated`), every storage bucket policy
3. `api/` and `api/_lib/` in full - every handler, its auth check, its input validation, what it does with the service role
4. `supabase/functions/` in full
5. `src/lib/` in full, then the components that render privileged surfaces (`Admin*`, `StaffDesk`, `Checkout`, `MyOrders`, `SubscriptionPanel`, `AuthModal`, `PasswordReset`) in full; the remaining components at a depth proportional to risk
6. `scripts/`, the dependency manifest and the lockfile
7. Cross-reference `STRIPE_INTEGRATION_TODO.md` and any other TODO-style document (Section 8)
8. Use `git log` to understand recent changes and any security-relevant history

### The filing bar - clear a finding against this before it goes in the file

Too much review output can be findings that are already decided, already documented, or not defects at all. Each one costs a real decision round to close. This is a filing bar, not a severity rule - a finding that fails it does not get downgraded, it does not get written.

Clear every finding against all three before filing:

**(a) Does the project already document this as deliberate?** Check `README.md`, `docs/`, code comments at the site, and the Settled positions above. If it does, the finding must **engage with that text** - quote it and say why it is wrong, out of date, or does not cover this case. A finding that restates a documented decision without mentioning that it does is not a finding.

**(b) Is the surface in a state the owner has declared?** Stubbed, demo-mode fallback, test mode, not yet launched. The consequences of a declared state are not separate findings. Report the state once if it is genuinely news; never enumerate its downstream symptoms. **But** a declared stub that is reachable in production, or that fails open, is a finding about the reachability, not the stub.

**(c) Is the "risk" already bounded by a fact stated elsewhere?** An RLS policy, a `SECURITY DEFINER` function's own ownership check, a Stripe-side constraint, a key that is public by construction (the Supabase anon key is browser-safe *only because* RLS constrains it - so verify the RLS before relying on the bound). A severity assigned without reading the bound is a severity assigned to the wrong thing.

---

**Verify against the path that actually runs in production.** The commonest failure mode in a review is not a missing control but a control verified only on the happy path or the dev path: a security header verified by reading config rather than requesting the deployed page, an RLS policy verified by reading SQL rather than querying as `anon`, a webhook verified with a valid signature but never with a missing or forged one, a price check verified with the client's own numbers. When assessing whether a control holds, exercise the hostile path and the production build (`npm run build`, then `npm run preview` or the deployed site) - and when you cannot, say so explicitly in the finding rather than recording it as verified.

---

## Section 1: Repository Hygiene Audit

The repository must have, at minimum:
- `README.md` - purpose, setup, run, deploy instructions
- `.env.example` - every environment variable the code reads, with no real values
- `.gitignore` - excluding env files, build output and dependencies
- A lockfile (`package-lock.json`)
- A documented way to apply the database schema (`supabase/migrations/` plus instructions)

Produce a table:

| Item | Present | Current | Notes |
|------|---------|---------|-------|
| README.md | ✓ | ✗ | Describes an Edge Function not in the tree |

"Current" means it agrees with the code at HEAD. Also record in this section:
- Whether any file that `.gitignore` excludes is nevertheless **tracked** (`git ls-files -ci --exclude-standard`). A `.gitignore` entry does not untrack a file that was committed before it was added.
- Whether `.env.example` lists every variable the code reads. Enumerate the reads rather than trusting the file: `grep -rhoE "(process\.env|import\.meta\.env|Deno\.env\.get\()\.?[A-Z_\"']+" api src supabase scripts | sort -u`.
- Whether a licence file exists, and whether `public/` assets carry any licensing or attribution requirement.

Flag missing or stale items as **Medium** findings unless a stale item misleads someone into an insecure setup, in which case file it at its real severity in the relevant section.

---

## Section 2: Security

This is the most critical section. Critical findings are blockers - they must be resolved before any other work proceeds.

Severity scale:
- 🔴 Critical - exploitable vulnerability with significant impact; fix immediately
- 🟠 High - serious risk; fix before next release
- 🟡 Medium - real issue but not immediately exploitable; fix in current sprint
- 🟢 Low - improvement; fix when convenient

### 2a. Authentication & Authorisation

- Does every `api/` handler that acts on behalf of a user verify the Supabase JWT server-side (signature and expiry, via `supabase.auth.getUser(token)` or equivalent) - not merely decode it, and not trust a user id supplied in the body?
- Does every admin-only handler and RPC check `is_admin()` (or equivalent) on the **server**? A check that exists only in a React component is not a check.
- Is there privilege escalation risk? Can a customer reach an admin RPC, a staff-desk RPC or a service-role code path through parameter manipulation, a missing check, or an RPC granted to `anon`/`authenticated` that should not be?
- **The staff desk's shared passphrase:** how is it hashed and compared, is it rate-limited or lockout-protected against guessing, is the comparison constant-time, is there any default or seeded passphrase that would be live on a fresh deployment, can it be rotated, and what does a passphrase holder see (PII scope)?
- Google OAuth: are redirect URLs restricted to known origins in the documented setup? Is there an open redirect through any `redirectTo` / `next` / return-URL parameter?
- Password reset and sign-up flows: can they be used for account enumeration or to hijack an account?
- **Guest orders:** how is a guest's access to their own order proven (token, email, session id)? Can one guest read or modify another's order by changing an identifier?
- Can a user of one account access another account's commits, orders, subscriptions, profile, Scentprint or chat transcript through any RPC, table, view or `api/` route?

### 2b. Input Validation & Injection

- Is all user-supplied input validated (type, length, range, enum) before it reaches the database, Stripe, Australia Post, Mailgun or Claude?
- SQL: is any SQL built by string concatenation in an RPC (`execute format(...)` with unquoted identifiers or `%s` on user input)? Are PostgREST filters built from user input in a way that lets a caller widen a query?
- Are there cases where user-supplied data reaches file system paths, storage object paths (bucket keys), shell commands (in `scripts/`), or HTML without escaping?
- **File uploads** (bottle images to the storage bucket, photographs in the Scent Memory flow): are they validated for MIME type, size and content - not just the filename or Content-Type header? Who can upload, and can an upload overwrite another object?
- Is there any `dangerouslySetInnerHTML`, `innerHTML`, or prerendered HTML (`scripts/prerender.mjs`) that interpolates catalogue, review or user content without escaping? Reviews and AI-generated copy are both user-influenced.
- Open redirects: any user-controlled URL passed to `window.location`, Stripe `success_url` / `cancel_url` / `return_url`, or an auth redirect?
- SSRF: any user-controlled URL fetched server-side (including image URLs handed to Claude)?
- Engraving text and other free-text order fields: length limits, character set, and how they reach fulfilment and labels.

### 2c. AI/LLM Security

The project calls Claude from `api/chat.ts`, `api/scent-ai.ts` and `api/conceive.ts`. This creates a specific attack surface:

- Is user-supplied content passed into prompts in a way that lets a user hijack the model's behaviour (prompt injection) - including indirectly, via catalogue copy, reviews or a shared Scentprint another user authored?
- Can prompt injection cause the model to leak the system prompt, other customers' data, pricing or stock logic, or to produce content that is then trusted downstream?
- Are LLM responses validated before use - parsed against the structured-output schema, checked before being written to the database (AI conception), and escaped before render? Does `scripts/check_output_schemas.mjs` actually cover every schema in use?
- Are the public AI endpoints (chat, scent AI) **unauthenticated**? If so, what bounds the cost: rate limiting, input size caps, `max_tokens`, model choice, caching? An open LLM proxy is a billing vulnerability.
- Is the admin-only conception endpoint actually admin-only on the server?
- Is `ANTHROPIC_API_KEY` used only server-side and never shipped in the client bundle? (Check the built `dist/` output, not just the source.)
- Are chat transcripts or uploaded photographs stored or sent to the model in a way that conflicts with the privacy notice?
- Is the model id current and are timeouts set so a hung model call does not hold a function to its maximum duration?

### 2d. Secrets & Credential Exposure

- Are any secrets, API keys, tokens or passwords hardcoded in source? Search broadly: `sk_live`, `sk_test`, `rk_`, `whsec_`, `sk-ant-`, `service_role`, `eyJ` (JWTs), `key-` (Mailgun), `password`, `secret`, `api_key`.
- **Are env files tracked?** `git ls-files | grep -i '\.env'`. Anything other than `.env.example` is a finding; establish what it contains (without reproducing values in the review) and whether the values are live.
- Scan git history for secrets ever committed, not just at HEAD: `git log --all --full-history --oneline -- "*.env" ".env*" "*.pem" "*.key" "*.p12"`, and `git log -p --all -S 'sk_live' -S 'whsec_' -S 'sk-ant-'` as a starting point. A secret removed in a later commit is still exposed in a public or shared repository and must be rotated.
- Is any `VITE_`-prefixed variable carrying a secret? Every `VITE_*` value is inlined into the public client bundle at build time.
- Is the Supabase **service-role** key ever reachable from the browser or logged?
- Are internal URLs, dashboard links or infrastructure details exposed in comments or config that should not be in source control?

### 2e. API Security Surface

- Is rate limiting in place on every public `api/` route, especially checkout, shipping quote, chat, scent AI, waitlist/marketing sign-up and anything that sends email? Vercel does not rate-limit for you.
- Is CORS configured restrictively on `api/` routes? Is any route returning `Access-Control-Allow-Origin: *` while also reading cookies or an Authorization header?
- Are HTTP security headers present on the deployed site and API responses? Required: Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options or `frame-ancestors`, Content-Security-Policy, Referrer-Policy. Check `vercel.json` and then **request the deployed pages** to confirm.
- Do error responses leak internal details - stack traces, Supabase/Postgres error text, Stripe error objects, file paths?
- Insecure Direct Object References: can a caller fetch or mutate another user's order, subscription, Stripe customer portal session, or shipment by changing an id?
- Mass assignment: is any request body spread directly into a database insert/update or a Stripe create call?
- Are HTTP methods enforced (a GET that mutates, a handler that accepts any method)?

### 2f. Payments Integrity (Stripe)

Money handling gets its own subsection because the batch model (authorise now, capture when the batch is met) is unusual and easy to get subtly wrong.

- **Price authority:** is every amount charged computed server-side from the database (fragrance, format, size, quantity, shipping, discount), or can the client influence it? Try it: tamper with the request body.
- **Webhook:** does `api/stripe/webhook.ts` verify the signature with `STRIPE_WEBHOOK_SECRET` against the **raw** body (body parsing disabled), reject unsigned or forged events, and handle replays and duplicate deliveries idempotently?
- Is the order/commit state changed only by verified webhook events or verified server-side retrievals - never by a client "payment succeeded" call such as a `confirm` route trusting its input?
- **Authorise-later lifecycle:** what happens when a manual-capture PaymentIntent's authorisation expires (typically 7 days) before the batch is met? Is there a capture/release path in the tree, or is it only described in the docs? What does a customer's card see in each outcome?
- Subscriptions: can a user hold more than one active subscription (`0032_one_active_subscription.sql`), cancel someone else's, or open another customer's billing portal?
- Idempotency keys on Stripe create calls; behaviour on double-submit.
- Is test-mode versus live-mode configuration unambiguous, and could a test key reach production or vice versa?
- `npm run test:security` (`scripts/test_checkout_security.mjs`): what does it actually assert, does it run against the real handlers, and does it pass?

### 2g. Dependency Security

- Run `npm audit --omit=dev --audit-level=moderate` and `npm audit --audit-level=moderate`. Report production-tree advisories at their real severity; dev-only advisories matter only where the tool runs in the build or on a developer machine.
- Are there packages with known vulnerabilities in the locked versions (`package-lock.json`), not just the ranges in `package.json`?
- Are devDependencies leaking into the production bundle or the serverless functions?
- Are the Edge Functions' remote imports (Deno URL imports or `npm:` specifiers) pinned to exact versions?

### 2h. Supabase Row-Level Security & Database Functions

This replaces a traditional server-side authorisation layer: with the anon key in every browser, RLS and RPC grants are the authorisation model. Read every migration in order and reason about the **final** state of each object, since later migrations alter earlier ones.

- Is RLS **enabled** on every table in the `public` schema (and any other exposed schema)? A table without RLS is readable and writable by anyone holding the anon key.
- Are there policies with `using (true)` or `with check (true)` on tables holding personal, order or payment data?
- Do write policies validate the row's ownership (`user_id = auth.uid()`) in `with check`, not just `using`?
- Every `SECURITY DEFINER` function: does it pin `search_path` (`set search_path = ''` or an explicit list), perform its own authorisation check, and validate its arguments? Who has `execute` on it - and has the default `PUBLIC` execute grant been revoked where it should be?
- Are there views that bypass RLS (views run as their owner unless `security_invoker` is set)?
- Storage buckets: public or private, and do the storage policies restrict who can upload, overwrite and delete?
- Can the counters that drive the batch model (`committed`, stock, oil inventory) be inflated or drained by a caller repeating an RPC, and are they protected against races?
- Verify, do not just read: if a Supabase project or a local Postgres with the migrations applied is available, query as `anon` and as a second ordinary user and attempt the reads and writes the policies are meant to prevent. Say which you did.

### 2i. Session & Data Handling

- Where is the Supabase session stored (localStorage by default)? Given that, is the XSS surface (2b) and the CSP (2e) strong enough to protect it?
- Is CSRF protection needed and present on any cookie-authenticated route?
- Is sensitive data (passwords, tokens, addresses, emails, card details) ever logged in `api/`, Edge Functions or the browser console?
- Is anything sensitive kept in `localStorage` (bag, orders, Scentprint, share codes) that would leak on a shared device?
- **Australian Privacy Act 1988 / APPs and Spam Act 2003:** the business sells to Australian customers (AUD, Australia Post). Is there a privacy notice reachable from every point of collection; is marketing consent captured explicitly, recorded, and honoured (`0014_profiles_marketing.sql`); does every marketing email carry a working unsubscribe; are photographs and chat transcripts covered by the notice; is there a retention or deletion path for customer data?

---

## Section 3: Performance

### 3a. Database Performance (Supabase / Postgres)

**Reads:**
- `select('*')` where only a few columns are needed, particularly on wide catalogue rows rendered in lists.
- Unbounded queries: list fetches without `.limit()` / `.range()` on tables that grow (orders, commits, reviews, chat transcripts, signals, waitlist).
- N+1 patterns: per-row queries inside loops in `api/`, Edge Functions or React effects, where a join, an `in()` filter or an RPC would do.
- Missing indexes on foreign keys and on columns used in `where`, `order by` and RLS predicates (`user_id`, `fragrance_id`, status columns). RLS predicates run on every row read, so an unindexed `user_id = auth.uid()` scales badly.

**Writes:**
- Unnecessary writes (no-op updates, writes on every render or keystroke).
- Multi-row operations done serially that could be a single statement or RPC.
- Triggers that recompute aggregates (`committed` reconciliation) with cost proportional to table size.

**Transactions & concurrency:**
- Are stock and batch counters updated atomically (single statement, row lock, or constraint), or is there a read-then-write race?
- Is any external I/O performed inside a transaction or a long-running RPC?

**Realtime:** if Supabase Realtime subscriptions are used, are they unsubscribed on unmount and scoped narrowly?

### 3b. Serverless / Backend Performance

- Cold-start weight: are heavy SDKs (Stripe, Anthropic, Supabase) imported only by the functions that need them?
- Are Claude calls given timeouts, `max_tokens` bounds, and streaming where the user waits on them? Is prompt caching used where a large stable prefix exists (the README claims it for `scent-ai.ts` - verify)?
- Are independent awaits run in parallel (`Promise.all`) rather than serially?
- Are cacheable responses (shipping quotes for a postcode, catalogue reads) cached, and are `Cache-Control` headers set appropriately - and never on personalised responses?
- Vercel function count is capped (see `scripts/check_function_count.mjs`): does route consolidation create a single function with too many responsibilities or an oversized bundle?

### 3c. Frontend Performance

- **Images:** `public/assets/` holds full-size photography. Check file sizes, dimensions versus rendered size, modern formats (WebP/AVIF), `width`/`height` attributes, `loading="lazy"` below the fold, and `srcset`. Measure the total weight of the homepage and a product page.
- **Bundle:** run `npm run build` and report the chunk sizes. Is code splitting used for admin, staff desk, checkout and the Scent DNA experience so shoppers do not download them? Are heavy dependencies pulled into the main chunk?
- Unnecessary re-renders: large inline style objects recreated each render, unstable props, missing memoisation on expensive derived data (matching/scoring in `formats.ts`, `scentdna.ts`).
- Data fetching: waterfalls on first load, duplicate fetches of the catalogue, missing loading states causing layout shift.
- Web fonts: `font-display`, preconnect, and how many families/weights load on first paint.
- Prerendering: does `scripts/prerender.mjs` produce correct per-page HTML, and does the SPA rewrite in `vercel.json` serve it?

---

## Section 4: Code Health & Cleanliness

### 4a. Code Quality

- Duplication: the same business rule (pricing, formats, shipping, catalogue shape) implemented in both `src/lib/` and `api/_lib/`, or in both TypeScript and SQL, where the two can drift. Prices and stock are the highest-risk duplicates.
- Dead code: exported symbols, components, routes, migrations-era RPCs or demo fallbacks that are never reached.
- Overly complex functions: very large components or functions with deep branching that are hard to reason about.
- Inconsistent error handling: some paths throw, some return error objects, some **silently fall back to seed/demo data**. A silent fallback that hides a production failure (for example, a failed Supabase read rendering demo stock as real) is a finding.
- Type safety: `any`, unchecked casts on API and database responses, `@ts-ignore` / `@ts-expect-error` without explanation.
- Magic numbers and strings: prices, MOQs, size lists, status strings and timeouts that should be named constants or come from the database.
- Commented-out code blocks.

### 4b. Architecture Consistency

- Is the split between client (`src/lib/`), serverless (`api/`), database (RPCs/RLS) and Edge Functions coherent? Is any logic that must be trusted running only on the client?
- Are the demo/offline fallbacks cleanly separated from the production path, so production cannot silently run in demo mode?
- Is routing consistent (hash routing vs path routing vs prerendered paths)?
- Are styling conventions consistent (inline style objects vs CSS files)?

### 4c. Testing

- What automated tests exist? Today the candidates are the scripts under `scripts/` (`test_checkout_security.mjs`, `check_output_schemas.mjs`, `check_function_count.mjs`). Run each and report the result.
- What coverage exists for the most critical paths: checkout price calculation, webhook handling, RLS policies, staff-desk access, subscription lifecycle, guest-order access?
- Are tests meaningful (asserting hostile inputs are refused) or smoke tests?
- Is there any test of the migrations themselves (applied in order to a clean database)?
- Are any checks skipped or disabled without explanation?

### 4d. Configuration & Environment Management

- Is configuration read consistently, and validated at startup (fail fast) rather than failing deep in a request?
- Does a missing server-side variable cause a function to **fail closed** (refuse) rather than fail open (skip a check, use a stub, or fall back to demo behaviour)?
- Hardcoded environment-specific values (URLs, postcodes, currency, email addresses) that should be environment variables.
- Is every variable documented in `.env.example` and `docs/SUPABASE_SETUP.md`'s reference table, and do the two agree?

### 4e. Logging & Observability

- Are errors in `api/` and Edge Functions logged with enough context to diagnose (route, order id, Stripe event id) and without sensitive data?
- Are critical paths logged: auth failures, admin actions, staff-desk access, payment state changes, webhook outcomes, AI calls and their cost drivers?
- Is there any error alerting (Vercel, Supabase, Stripe dashboard alerts, an error tracker)? Would anyone notice if the webhook started failing?
- Is analytics (`src/lib/analytics.ts`, GA4) gated on consent where required, and does it avoid sending PII?

---

## Section 5: Documentation

**README.md completeness:**
- Does it clearly explain what this project is?
- Does it explain local setup from a fresh clone, including the demo mode versus a real Supabase project?
- Does it explain how to run, build, lint and test?
- Does it explain how to deploy (Vercel, Supabase migrations, Edge Functions, Stripe webhook registration)?
- Is it current? Diff every file, component, function and migration it names against `git ls-files` and report each mismatch.

**Code documentation:**
- Are complex or non-obvious sections explained (the batch model, the matching maths, the reconciliation trigger)?
- Are workarounds, stubs and known limitations commented at the site?

**API documentation:**
- Are the `api/` routes documented with method, auth requirement, request and response shape?
- Are the database RPCs documented with who may call them and what they check?

**Data documentation:**
- Are the table structures, ownership rules and RLS intent documented?
- Is the migration order and the process for adding a migration documented?

**Operational documentation:**
- `docs/SECURITY_SHOPPER_FLOW_ROLLOUT.md` and `STRIPE_INTEGRATION_TODO.md`: are they current, and is it clear which steps are done?

---

## Section 6: Dependency & Supply Chain Health

- Is `package-lock.json` committed and in sync with `package.json` (`npm ci` succeeds cleanly)?
- Are there unused dependencies, or used-but-undeclared ones?
- Is the Node.js version declared (`engines`, `.nvmrc`) and does it match what Vercel builds with?
- Are the Python helper scripts' dependencies declared anywhere?
- Are dependency ranges appropriate (`^` on everything means each fresh install can differ from the lockfile's tested set)?
- Is anything vendored or forked locally?
- Is there automated dependency update tooling (Dependabot, Renovate)? If not, note it once.

---

## Section 7: Operational & Deployment Readiness

- Is the deployment process documented and reproducible end to end: Vercel project and env vars, Supabase migrations, Edge Function deploys and secrets, Stripe webhook endpoint registration, DNS?
- Is there a health or status check that verifies the database and critical dependencies, and is anything monitoring it?
- Are secrets injected at runtime from the platform (good) or present in the repository or build output (bad)? Inspect `dist/` after a build for leaked server-side values.
- **Migrations:** are they idempotent or guarded, is there a rollback story, and are seed migrations safe to run against a production database that already holds real data?
- **Backups:** is the Supabase backup / point-in-time recovery configuration documented, and has a restore ever been rehearsed?
- Are there runbooks for the predictable failures: webhook failing, Stripe authorisations expiring before a batch is met, Australia Post API unavailable, Claude unavailable, email delivery failing?
- Is the Vercel plan's function limit, duration limit and body-size limit respected by every route?

---

## Section 7b: Project Gates & Toolchain Conformance

The project encodes several of its own rules as build-time gates in `package.json`'s `build` script and in `scripts/`. **For each one, check both that the rule holds AND that the gate actually runs** - a gate that exists but is not wired in, is bypassed by the deployment build, or is permanently failing and therefore ignored, is itself the finding.

- `npm run build` runs `tsc -b`, `scripts/check_function_count.mjs`, `scripts/check_output_schemas.mjs`, `vite build` and `scripts/prerender.mjs`. Run it from a clean clone (`npm ci` first). Does it pass? Does Vercel run this same `build` script, or a different command that skips the gates?
- `npm run lint`: does it pass? Is it run anywhere automatically?
- `npm run test:security`: does it pass, and is it run before deploy?
- **Is there any CI** (`.github/workflows/`, Vercel checks)? If not, every gate above runs only when a person remembers to. State that once, plainly.
- Is there a pre-commit or pre-push secret scan? Given Section 2d, its absence matters more here than usual.
- `tsconfig*.json`: is `strict` enabled for both the app and the API? Are `api/` and `supabase/functions/` type-checked by the build at all?

---

## Section 8: TODO Cross-Reference

Read `STRIPE_INTEGRATION_TODO.md`, any TODO/FIXME comments in the code (`grep -rn "TODO\|FIXME\|XXX\|HACK" api src supabase scripts`), and any open-steps lists in `docs/`. When any finding in this review matches or relates to one of those items, note the cross-reference inline at the end of that finding using this format:

See also: STRIPE_INTEGRATION_TODO.md - "relevant item text"

Do not enumerate TODO items in the output. Only surface them when they are directly relevant to a finding you have already made independently. A TODO that describes a **security control as still to be done** while the code path it would protect is already live is itself a finding.

---

## Section 8b: Reviewer's Discretion

Use your judgment to add any additional sections or findings not covered above. This review is intended to be exhaustive. If you identify a pattern, risk, or opportunity that does not fit the categories above, create a new section for it.

Consider these additional areas (not exhaustive):
- Accessibility (a11y) - does the storefront meet WCAG 2.1 AA (contrast on the dark palette, keyboard navigation of drawers and modals, focus management, alt text, form labels)?
- Consumer law - Australian Consumer Law obligations on pricing display (GST-inclusive prices), refunds and returns, and pre-order / batch terms that a customer agrees to before a card is held.
- Product claims - "Inspired by <brand>" attributions and use of third-party fragrance brand names and imagery: is the wording and asset use consistent with comparative-advertising and trade mark norms? File as Confidence: Low unless clear-cut; this is a legal question, not a code one.
- SEO and structured data - prerendered titles, descriptions, canonical URLs, sitemap, Product JSON-LD (`image`, and on the Offer `hasMerchantReturnPolicy` and `shippingDetails`). Never recommend adding `aggregateRating` or `review` markup unless real, collected reviews back it.
- Email deliverability - SPF, DKIM, DMARC for the Mailgun sending domain.
- Technical debt - patterns creating compounding debt that should be addressed proactively.

---

## Output Format

Structure the saved file as follows. Use this exact structure.

---

# Maison Obsidian Code Review - YYYY-MM-DD

**Reviewer:** Claude Opus
**Review framework:** Adam Burgess - https://github.com/qant-au
**Date:** YYYY-MM-DD
**Scope:** Maison Obsidian repository at commit `<short hash>` (`git log -1 --format=%h`)
**Prior reviews consulted:** [List any found, or "None - this is the first review"]

---

## Executive Summary

[5-8 paragraphs giving an honest overall assessment of the project's health. Lead with the most critical security and payments findings. Summarise the key themes across all categories. Be direct - if something is in poor shape, say so. Do not pad this section.]

---

## Section 1: Repository Hygiene Audit

[Table as specified above]

**Summary:** [The highest-priority gaps.]

---

## Section 2: Security

Priority key: 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low

### 2a. Authentication & Authorisation
### 2b. Input Validation & Injection
### 2c. AI/LLM Security
### 2d. Secrets & Credential Exposure
### 2e. API Security Surface
### 2f. Payments Integrity (Stripe)
### 2g. Dependency Security
### 2h. Supabase Row-Level Security & Database Functions
### 2i. Session & Data Handling

For each finding use this format:

**[area] - Brief title of finding**
Severity: 🔴 Critical / 🟠 High / 🟡 Medium / 🟢 Low
Location: path/to/file.ts:line_number (omit if not file-specific)
Description: What the issue is and why it matters in concrete terms.
Verified by: What you exercised (request sent, query run, build inspected), or "Read only - not exercised" with the reason.
Recommendation: Specific, actionable steps to fix it.
Status: New | Recurring (code-review-YYYY-MM-DD.md)
See also: STRIPE_INTEGRATION_TODO.md - "item text" (only if applicable)

`[area]` is one of: `api`, `database`, `edge-function`, `frontend`, `scripts`, `config`, `docs`, `assets`.

---

## Section 3: Performance

### 3a. Database Performance
### 3b. Serverless / Backend Performance
### 3c. Frontend Performance

[Same finding format as Section 2]

---

## Section 4: Code Health & Cleanliness

### 4a. Code Quality
### 4b. Architecture Consistency
### 4c. Testing
### 4d. Configuration & Environment Management
### 4e. Logging & Observability

---

## Section 5: Documentation

---

## Section 6: Dependency & Supply Chain Health

---

## Section 7: Operational & Deployment Readiness

---

## Section 7b: Project Gates & Toolchain Conformance

---

## Section 8b: [Additional sections at reviewer's discretion]

---

## Section 9: Carried-Forward Items - Verification Results

One subsection per prior action-items file that still had open items. For each open item: its original ID, title, and the verdict of re-checking it at HEAD - **Still open** (with the new item number it becomes in today's action-items file), **Since fixed** (with evidence), or **No longer applicable** (with reason). Close with a count line: "N items re-checked: X carried forward, Y closed as done, Z ruled out."

On the first cycle, write: "None - this is the first review."

---

## Appendix: Finding Summary Table

Complete list of all findings across Sections 1-9, sorted by Severity (Critical to Low) then section.

| # | Area | Section | Severity | Title | Status |
|---|------|---------|----------|-------|--------|
| 1 | database | Security 2h | 🔴 Critical | ... | New |

Status values: `New`, `Recurring (code-review-YYYY-MM-DD.md)`, or `Carried forward (action-items-YYYY-MM-DD.md #N)`.

**Total findings:** X (Critical: N, High: N, Medium: N, Low: N)

---

## Action Items Format

After saving the review file, create a second file `reviews/action-items-YYYY-MM-DD.md`. This file distils every finding in the review into independently-actionable work items, ordered by execution priority: Critical first, then High, then Medium (grouped by theme: Security hardening, Payments, Performance, Architecture & code health, Testing, Configuration & operational, Documentation & dependency hygiene), then Low. Omit any finding that is purely informational and produces no action.

Use this exact structure:

---

# Maison Obsidian Action Items - YYYY-MM-DD

Derived from [code-review-YYYY-MM-DD.md](./code-review-YYYY-MM-DD.md). Review framework by Adam Burgess - https://github.com/qant-au. Items are listed in execution order: Critical first, then High, then Medium grouped by theme, then Low. Each item is independently actionable and references its finding in the review by appendix number (e.g. `#4` -> row 4 of the Appendix Finding Summary Table) and originating section.

**This is the single working action-items document.** Every prior `action-items-*.md` (all of them in `reviews/history/`) is now fully closed - items still open in them were re-verified at HEAD and carried into this file. Do not re-open the older files; add new work here.

Priority key: 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low

---

## Carried forward from prior cycles

| Source | Now | Title | Why still open |
|--------|-----|-------|----------------|
| action-items-YYYY-MM-DD.md #N | #M | ... | ... |

**N items carried forward** (X Critical, Y High, ...). A further Z prior open items were closed during this review's verification pass - see Section 9 of the review for the evidence.

[Omit this section on the first cycle.]

---

## 🔴 Critical (do first)

### 1. [Brief title]
- **Why:** [Concrete reason - what breaks or what risk exists without this fix]
- **What:** [Specific, actionable steps to resolve]
- **Where:** `path/to/file.ts:line_number` (omit if not file-specific)
- **Refs:** Review #N (Section X.y)

[Repeat for each Critical finding]

---

## 🟠 High (do before next release)

### N. [Brief title]
- **Why:** ...
- **What:** ...
- **Where:** ...
- **Refs:** Review #N (Section X.y)

---

## 🟡 Medium - [Theme group name]

### N. [Brief title]
- **Why:** ...
- **What:** ...
- **Where:** ...
- **Refs:** Review #N (Section X.y)

[Repeat for each Medium theme group]

---

## 🟢 Low (do when convenient)

### N. [Brief title]
- **What:** ...
- **Where:** ...
- **Refs:** Review #N (Section X.y)

---

*End of action items. See [code-review-YYYY-MM-DD.md](./code-review-YYYY-MM-DD.md) for full context, severity definitions, and the executive summary.*

---

Rules for writing action items:

- **Every finding with a Recommendation becomes one action item.** Do not merge unrelated findings into one item. Do not split a single finding into multiple items unless the finding explicitly identifies separable independent steps.
- **Ordering within a priority band:** sort by section number, then by area (database and api before frontend).
- **Why is mandatory** on Critical and High items. For Medium and Low it may be omitted when the title is self-explanatory.
- **Where** should include the most specific location available: file path + line number from the finding. If the finding spans multiple files, list them all. If no specific file was identified, omit the field entirely.
- **Refs** must include the appendix row number (`#N`) and the section code (e.g. `Section 2h`, `Section 3a`). If a finding has a TODO cross-reference from Section 8, include it here too.
- **Recurring findings** should be flagged with `(recurring - first seen code-review-YYYY-MM-DD.md)` appended to the Refs line.
- **Carried-forward items** keep their original Why/What (updated for any code movement) and append `(carried forward from action-items-YYYY-MM-DD.md #N)` to their Refs line. They sort into the priority bands on their **current** severity, not the severity they were originally filed at.
- **Confidence: Low findings** from the review should be included but annotated: add `_Confidence: Low - [reason from review]_` as a final line in the item.
- **A leaked credential is always its own Critical item** whose What starts with "Rotate", before any clean-up of the repository or its history. Removing a secret from the tree does not un-leak it.
- The action items file is a separate deliverable from the review. Do not reproduce the full finding text - only the information needed to act on it.
- **After writing this file, close out the prior files** per the carry-forward protocol, then re-run the open-item count and confirm every file except today's reports `open=0`. That verification is part of the deliverable, not optional.
- **Then archive the superseded pair into `reviews/history/`** per "Archive the superseded results files" above. This is the final step of the cycle.

---

## Final Instructions

- Be thorough. Read files in full. Do not skim and assume.
- Be specific. File paths and line numbers wherever possible.
- Be honest. If an area is in excellent shape, say so clearly. If it is poorly maintained, say that too. The purpose of this review is to produce an accurate picture, not a polished one.
- Do not pad findings. Only include real issues - not hypothetical risks or trivial style preferences.
- When uncertain whether something is a genuine issue, include it clearly marked as: Confidence: Low - explain why you are uncertain, and what would need to be confirmed to resolve the uncertainty.
- **Do not reproduce secret values** in either output file. Name the file, line, variable and key type; never the value.
- **Do not record a control as verified when you only read it.** Say which path you exercised in the `Verified by:` line.
- **Do not act against live systems.** Exercising a control means local builds, a local or disposable database, test-mode Stripe and requests against the site that a normal visitor could make. Never place a live order, capture or refund a real payment, send email to real customers, or modify production data. If confirming a finding would require any of those, stop at "Read only - not exercised" and say what would confirm it.
- **A grep hit is not a finding, and a grep miss is not a clean result.** Code frequently documents its decisions, and sometimes the *absence* of a thing, in the same vocabulary as the code itself. Confirm every grep hit by reading the surrounding lines, confirm every grep miss by opening the files you expected to match, and prefer structural patterns (`create policy`, `security definer`, `grant execute`, `export default async function`, `capture_method`) over bare keywords.
- **A loop that prints nothing is not a clean result** until you have confirmed it ran. Under zsh, an unquoted variable is not word-split (`for f in $files` iterates once over the whole string); use an array, `${=VAR}`, or run the block under `bash -c`.
- The output file is the authoritative record of this review. Future reviews will reference it. Make it worth reading.
- Save both files when you have completed all sections. Do not save either file incrementally. Write each file in a single operation at the end. Save the review file first, then the action items file.
- **Then close out the prior action-items files** per the carry-forward protocol, and re-run the open-item count so the record shows every file except today's at `open=0`. The review is not complete until there is exactly one open action-items list.
- **Finally, archive the superseded results pair into `reviews/history/`** and repoint any path reference the move breaks. `reviews/` must end the cycle holding only `code-review-instructions.md`, today's two files, and `history/`.
