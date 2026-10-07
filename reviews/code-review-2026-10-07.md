# Maison Obsidian Code Review - 2026-10-07

**Reviewer:** Claude Opus
**Review framework:** Adam Burgess - https://github.com/qant-au
**Date:** 2026-10-07
**Scope:** Maison Obsidian repository at commit `6054bbb`
**Prior reviews consulted:** None - this is the first review

---

## Executive Summary

Maison Obsidian is in better shape than most storefronts of its age. The parts that handle money are the strongest parts of the codebase: every line in a bag is priced server-side from the live catalogue, postage is re-quoted from Australia Post before the charge, the Stripe webhook verifies its signature against the raw body with the body parser disabled, and paid orders are written through a `SECURITY DEFINER` RPC that takes an advisory lock per Checkout Session and refuses replays. Migration `0029_checkout_security.sql` closed the public insert policy on `commits`, revoked the client-side billing RPCs and moved the staff desk from a shared passphrase to account membership. I probed the production database as an anonymous visitor and every customer table (`commits`, `subscribers`, `scent_profiles`, `stripe_customers`, `customer_profiles`, `marketing_signups`, `order_fulfilment`) returned nothing; `waitlist` and `staff_members` are closed at the grant level; the `reviews.user_id` column is denied. Row-Level Security is doing its job.

The most serious problems are not in the code paths that charge cards. They are in the deployment around them. The production deployment carries **no Content-Security-Policy, no X-Frame-Options, no X-Content-Type-Options and no Referrer-Policy** on any response, while the Supabase session sits in `localStorage` where any injected script can read it. The production environment has a **Supabase secret key in the `SUPABASE_ANON_KEY` slot**, and the project's own unauthenticated `/api/stripe/status` endpoint tells anyone who asks exactly that, along with the fact that Stripe is in live mode and the postcode the business posts from. The live site **collects names, addresses, phone numbers, photographs and chat transcripts, and takes payments, with no privacy policy, no returns policy and no support contact published** - the help page says as much in its own words, and the bag, checkout and concierge all promise "30-day returns" against a policy that does not exist.

The database carries two authorisation gaps worth fixing before they matter. The guest-order read policy added in `0018` matches unclaimed orders to the **email claim in the caller's JWT with no check that the address was ever confirmed**, so if Supabase's email confirmation is off (the project's own setup guide suggests turning it off for testing) anyone can sign up as a guest's address and read their delivery details. The `0033` reviews migration shows the authors know the right test - it checks `email_confirmed_at` - but the earlier policy was never brought into line. Separately, "VIP-only" is enforced nowhere on the server: checkout does not check `vip_only`, and both the `enroll_subscriber` RPC and a leftover `with check (true)` insert policy let any visitor make themselves a VIP for free.

The public AI endpoints are an open Claude proxy with an attacker-controlled system prompt. `api/chat.ts` and `api/scent-ai.ts` take the catalogue text **from the request body** and place it in the system prompt, the only rate limit is a per-warm-instance in-memory counter the README itself describes as best-effort, and `scent-ai` will send up to 7 MB of image and 8,192 output tokens to `claude-opus-5` per unauthenticated call. The `create-shipment` Edge Function has no authorisation check at all: any holder of the public anon key can create shipment rows for any commit and, if the Australia Post credentials were ever set, buy labels.

The documentation has drifted badly from the code. The README still describes an authorise-now-capture-later batch model, a `capture-batch` Edge Function, a shared staff passphrase, Mailgun as the mail provider and a migration list that stops at 0024; none of those are true at HEAD. The code sends mail through Resend using environment variables that appear in no documentation and no `.env.example`, and when they are unset the abandoned-checkout reminder and launch-day email are silently skipped. The review instructions' own roles section inherited the stale passphrase description and has been corrected in this cycle.

Operationally the project runs on trust. There is no CI, no pre-commit hook, no Dependabot, no health check beyond a config echo, no alerting and no runbooks. The build's type-check runs the non-strict `tsconfig.json` and never touches `api/`; the strict `tsconfig.app.json` is unreferenced and currently fails. `.env` is committed despite being gitignored (it holds only the publishable key, so nothing has leaked) and the production browser bundle appears to get its Supabase configuration from that committed file rather than from Vercel. Nine demo migrations seed inventory and prices in ways that would overwrite real data if re-run, and the live database still carries the demo stock and oil figures from `0003` and `0006`, which the storefront presents to shoppers as "Ready to ship".

---

## Section 1: Repository Hygiene Audit

| Item | Present | Current | Notes |
|------|---------|---------|-------|
| README.md | ✓ | ✗ | Describes authorise-later payments, a `capture-batch` Edge Function, a shared staff passphrase, Mailgun, `MyReservations.tsx`, `LayoutSwitch.tsx` and a migration list ending at 0024. None exist at HEAD. See Section 5. |
| .env.example | ✓ | ✗ | Lists `MAILGUN_*` (never read by any code) and omits `RESEND_API_KEY`, `RECOVERY_EMAIL_FROM`, `RECOVERY_REPLY_TO`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `ANTHROPIC_KEY` and the Edge Function's `AUSPOST_API_KEY` / `_PASSWORD` / `_ACCOUNT_NUMBER` / `_PRODUCT_ID` / `_FROM_*`. Enumerated with the grep in the instructions. |
| .gitignore | ✓ | ✓ | Excludes `.env`, `.env.*`, `dist`, `node_modules`. |
| Tracked-but-ignored files | - | ✗ | `.env` **is tracked** (`git ls-files -ci --exclude-standard` returns it). Committed in `7f29390` ("Rename .env.example to .env", 2026-07-02), before the ignore rule. It holds `VITE_SUPABASE_URL`, the `sb_publishable_` anon key, an empty `VITE_STRIPE_AUTHORIZE_URL` and an empty `ANTHROPIC_API_KEY`. No secret value has ever been committed to it (every revision inspected). See 2d. |
| package-lock.json | ✓ | ✓ | `npm ci` installs cleanly (195 packages). |
| Schema application documented | ✓ | partial | `docs/SUPABASE_SETUP.md` §3 lists 0001-0007 in a table and says "every migration except 0002 is safe to re-run", which is false for 0003, 0006, 0021, 0027 and 0028 (Section 7). |
| LICENSE | ✗ | - | No licence file. Repository is private; the `public/assets/` photography carries no attribution or licence note and 24 of the files are named after third-party brands and products (Section 8b). |
| Node version declared | ✗ | - | No `engines` field, no `.nvmrc`. Local build ran on Node 26; Vercel's default is whatever the project setting says. |
| Python helper deps | ✗ | - | `scripts/import_catalogue.py` and `scripts/trim_bottle_renders.py` name `openpyxl` and `pillow` in docstrings only; no `requirements.txt`. |

**Summary:** The two documents a new engineer reads first (README, `.env.example`) would both lead them to configure the wrong things: Mailgun instead of Resend, a staff passphrase that no longer works, and a `VITE_STRIPE_AUTHORIZE_URL` that nothing uses. `.env` must be untracked. The committed values are public by construction, so there is nothing to rotate.

---

## Section 2: Security

Priority key: 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low

### 2a. Authentication & Authorisation

**[database] - Guest-order read policy trusts the JWT email claim without a verification check**
Severity: 🟠 High
Location: supabase/migrations/0018_guest_orders.sql:11-21
Description: `commits_select_own` grants a signed-in user every row where `user_id is null and lower(user_email) = lower(auth.jwt() ->> 'email')`. The email claim in a Supabase JWT is whatever address the account was created with; it is only trustworthy if the project requires email confirmation before issuing a session. `docs/SUPABASE_SETUP.md` §5 explicitly suggests turning *Confirm email* off ("so sign-ups log in immediately"). With it off, anyone who learns a guest's email (receipts, a shared order reference, a data breach elsewhere) can sign up as that address and read the guest's name, street address, phone number, engraving and what they bought. The project already knows the correct test: `can_review()` in `0033_reviews.sql:60-64` requires `auth.users.email_confirmed_at is not null` before it will match a guest order by email. The `0018` policy was never updated to match, and `store.ts:fetchMyCommits` builds its own `or()` filter on the same assumption.
Verified by: Read only - not exercised. Confirming it would require creating an account on the production project under another person's address. Whether email confirmation is currently enforced is a dashboard setting I cannot read; the finding stands regardless because the policy should not depend on it.
Recommendation: Rewrite the second branch of `commits_select_own` to join `auth.users` and require `email_confirmed_at is not null`, exactly as `can_review` does (or expose a `SECURITY DEFINER` `my_orders()` RPC that does the check and drop the email branch from the policy). Add a note to `SUPABASE_SETUP.md` that *Confirm email* must be **on** in production and why.
Status: New

**[database] - VIP-only is not enforced on the server, and VIP enrolment is free self-service**
Severity: 🟡 Medium
Location: api/_lib/catalogue.ts:112-117 (`buyable` ignores `vipOnly`); api/stripe/checkout.ts:32; api/stripe/subscribe.ts:39-40; supabase/migrations/0001_init.sql:167-188 and 211-213; src/App.tsx:111-114 and 352-359
Description: The only server-side VIP gate was inside `commit_to_batch`, which `0029` revoked. `priceLines()` now sells a `vip_only` fragrance to anyone who POSTs its id; `subscribe.ts` in "choose" mode accepts a `vipOnly` or format-`hidden` fragrance (only the surprise pool filters them). The client hides VIP bottles behind `frag.vipOnly && !vip` in `App.tsx:111`, which is not a check. Separately, `enroll_subscriber(p_email, p_tier default 'vip')` is granted to `anon` and `authenticated` and `App.tsx:joinVip` calls it with no payment, and the `0001` policy `subscribers_insert ... with check (true)` was never dropped, so a visitor can also insert `{email, user_id: <any uuid>, tier: 'vip'}` directly. The concierge system prompt tells customers the VIP Club costs "$120/yr" (`api/chat.ts:29`); there is no path that takes that money.
Verified by: Read only - not exercised. Exercising it would place a live order or write to the production `subscribers` table.
Recommendation: Decide whether VIP is a product. If it is: check `vip_only` against `subscribers` in `priceLines()`/`subscribe.ts` using the service client, drop `subscribers_insert`, and gate `enroll_subscriber` behind a paid Stripe product. If it is not: remove the `vip_only` flag from the catalogue, the VIP copy from the concierge prompt and the `/about` enrolment, and record the position in the instructions file.
Status: New

**[edge-function] - `create-shipment` performs no authorisation and would buy labels for any caller**
Severity: 🟠 High
Location: supabase/functions/create-shipment/index.ts:133-186
Description: The handler reads `commitId` and `shipTo` from the body, looks the commit up with the service-role client and inserts a `shipments` row, calling the Australia Post Shipping API (a paid label purchase) when `AUSPOST_API_KEY`/`_PASSWORD`/`_ACCOUNT_NUMBER` are set. There is no `is_admin()` check, no check that the caller owns the commit and no rate limit. Supabase Edge Functions accept any valid JWT by default, and the anon key is in every browser. Nothing in `src/` calls this function (`adminCreateShipment` uses the `admin_create_shipment` RPC), so it is dead from the app's point of view but live if it was ever deployed. The remote import `https://esm.sh/@supabase/supabase-js@2` is pinned to a major only.
Verified by: Read only - not exercised. I did not invoke the production function; whether it is deployed is not visible from the repository.
Recommendation: Either delete the function and the `docs/SUPABASE_SETUP.md` §7 instructions for it, or add a server-side admin check (verify the bearer token with the anon client and call `is_admin`, as `api/_lib/stripe.ts:isAdminRequest` does), restrict `shipTo` to the commit's own stored address, pin the import to an exact version, and run `supabase functions list` on the project to confirm whether it is currently deployed.
Status: New

**[database] - Earlier `SECURITY DEFINER` functions keep the default `PUBLIC` execute grant**
Severity: 🟢 Low
Location: supabase/migrations/0001_init.sql through 0024_scent_signals.sql (every `create function` without a matching `revoke ... from public`)
Description: `0025` documents the problem exactly ("Postgres grants EXECUTE to PUBLIC by default, and revoking from anon while PUBLIC still holds it changes nothing") and every migration from 0025 on revokes first. The twenty-odd functions created before it - `admin_*`, `set_subscription_pick`, `set_subscription_mode`, `cancel_subscription`, `claim_scentprint`, `draw_subscription_scent`, `new_scent_code`, `commit_size_counts` - still carry the default grant, so `anon` can call them. Each one checks `is_admin()` or `auth.uid()` internally, so the exposure is bounded, but the bound is one forgotten `if` away from not holding.
Verified by: Read only (grants read from the migrations in order).
Recommendation: One migration that `revoke execute ... from public` on every function in the schema and re-grants to the intended roles. Supabase's `supabase db lint` reports these as `function_search_path_mutable`/`public execute` warnings and can confirm the list.
Status: New

**[database] - `cancel_subscription` RPC still callable by the owner directly, bypassing Stripe**
Severity: 🟡 Medium
Location: supabase/migrations/0012_subscriptions.sql:118-133; src/lib/subscription.ts:270-286
Description: `cancel_subscription(uuid)` marks a row `cancelled` and is granted to `authenticated`. The client's `cancelSubscription()` calls `/api/stripe/cancel-subscription` first, but if that call returns `null` (network error, non-JSON response, 404) it **falls through to the RPC** and marks the row cancelled with the Stripe subscription still active. The account page then shows "Cancelled" while Stripe keeps charging the card every month; `customer.subscription.deleted` never arrives because nothing deleted it. A user can also call the RPC directly through PostgREST with the same effect.
Verified by: Read only - not exercised.
Recommendation: Revoke `cancel_subscription` from `authenticated` (keep it for `service_role`/admin) and make the client path fail loudly when the Stripe route is unreachable rather than falling back to the DB write. The cancellation of a Stripe-billed subscription must only ever be recorded by the webhook or by the route that just cancelled it at Stripe.
Status: New

Google OAuth and password reset were checked and are sound: both `redirectTo` values are `window.location.origin` (no user-controlled URL), reset responses are deliberately non-enumerating (`auth.ts:101-110`), and the recovery session is forced through `PasswordReset` before use.

### 2b. Input Validation & Injection

**[api] - Engraving and free-text order fields have length limits but no character-set limit**
Severity: 🟢 Low
Location: api/_lib/stripe.ts:223 (`engraving?.trim().slice(0, 28)`); api/stripe/checkout.ts:52-56
Description: Engraving (28 chars), delivery name (120), phone (40), notes (450) and address lines are length-capped and otherwise passed through to Stripe product descriptions, the staff desk, the pack list and the CSV export. Unicode control characters, bidi overrides and zero-width characters survive and would print on labels. The CSV export quotes correctly (`staffDesk.ts:222-228`) so formula injection via a leading `=` is mitigated by quoting but not neutralised (Excel still evaluates `="..."` content in some versions).
Verified by: Read only.
Recommendation: Normalise to NFC and strip control/format characters (`\p{C}`) in `priceLines()` and `checkout.ts`; prefix CSV cells starting with `=`, `+`, `-`, `@` with a `'`.
Status: New

No `dangerouslySetInnerHTML`, `innerHTML` or `eval` exists in `src/` (grep over the tree, confirmed by reading the rendering paths). `scripts/prerender.mjs` and `src/lib/snapshot.ts` escape every interpolated catalogue field, and `headTags()` escapes `<` inside the JSON-LD script. Review bodies render through React text nodes. Image uploads are validated by header (`conceive.ts:inspectImage`) and bucket MIME/size limits, and only admins can write to the bucket. PostgREST filters built from user input (`store.ts:138`) quote the email; RLS bounds the result anyway. No SSRF: the only server-fetched URLs are fixed Australia Post, Stripe, Resend and Anthropic hosts; the Scent Memory photograph arrives as base64, not a URL.

### 2c. AI/LLM Security

**[api] - The public AI routes accept their system prompt from the request body**
Severity: 🟡 Medium
Location: api/chat.ts:36-47 and 136 (`buildSystem(body.catalogue, body.profile)`); api/scent-ai.ts:403 and 415 (`cacheableSystem(str(body.catalogue, 24_000))`)
Description: Both unauthenticated routes build the system prompt from a `catalogue` string (8,000 and 24,000 chars respectively) and, for chat, a `profile` string (1,200 chars) that the **client** supplies. The comments explain why (the storefront's lexicon is the source of truth; the string is deterministic so it caches), but the effect is that any caller can replace the system prompt with their own instructions and use the route as a general-purpose Claude proxy. The structured-output schemas on `scent-ai` constrain the *shape* of the answer, not its content, and `chat` streams free text. Combined with the per-instance rate limit below, the cost and reputational exposure is real: the model answers in the house's name on the house's bill.
Verified by: Read only - not exercised against production (sending adversarial prompts would spend the owner's Anthropic credit).
Recommendation: Build the catalogue context on the server from `loadCatalogue()` (the route already has the anon client; `catalogueContext()`/`catalogueSummary()` are pure functions that can move to `api/_lib/`) and ignore any client-supplied `catalogue`. Fetch the taste `profile` server-side from the bearer token rather than trusting the body. Keep the cache breakpoint; it works just as well on a server-built string.
Status: New

**[api] - Open LLM endpoints are bounded only by a per-warm-instance in-memory counter**
Severity: 🟡 Medium
Location: api/chat.ts:50-78; api/scent-ai.ts:53-77; api/conceive.ts:135-159
Description: Each route keeps a `Map` of hits per IP in module scope. The README says plainly that this "bounds abuse per warm instance rather than globally - for strict distributed limits, back it with Upstash/Redis or a Supabase table." That caveat was written for a demo; the site is now live with `sk_live` Stripe keys. Vercel scales to many instances and recycles them, so the limit resets constantly, and `x-forwarded-for` is the first value in a client-settable header chain. The cost drivers per call are substantial: `scent-ai` uses `claude-opus-5` with up to 8,192 output tokens and accepts a 7 MB base64 image; `chat` streams up to 1,024 tokens from `claude-opus-4-8`; a server-side fallback model is enabled on each.
Verified by: Read only - not load-tested.
Recommendation: Move the counter to a shared store (an Upstash Redis or a small Supabase table keyed by IP and minute), add a daily cap per IP and a global daily spend ceiling, require a Supabase session for the `imagine` (image) operation, and set `maxDuration` and an Anthropic client `timeout` on each route so a slow call cannot hold a function open. Set a hard-stop alert on Anthropic spend.
Status: New

**[api] - Upstream error text returned to unauthenticated callers**
Severity: 🟢 Low
Location: api/scent-ai.ts:361-368 and 442-443; api/conceive.ts:240-247; api/marketing.ts:54-61
Description: `apiDetail()` returns up to 300 characters of the Anthropic API's own error message to the caller on any `APIError`. The comment argues it "describes the request, so there is nothing sensitive in it". That is usually true, but it also reveals model ids, beta flags and schema details, and it is sent to anonymous callers on `scent-ai`.
Verified by: Read only.
Recommendation: Log the detail server-side and return it only when `isAdminRequest()` passes, as `api/_lib/stripe.ts:route()` already does for the Stripe routes.
Status: New

Positives, verified: `ANTHROPIC_API_KEY` appears nowhere in `dist/` after a production build (grep for `sk-ant-`, `service_role`, `whsec_`, `sk_live`, `sk_test`, `sb_secret_` over `dist/` is empty). `conceive` and `marketing` check `is_admin()` server-side via the bearer token. Model outputs are normalised and clamped before use (`conceive.ts:normalise`, `scentai.ts:toScentprint`), anything naming a fragrance outside the candidate set is dropped, and `scripts/check_output_schemas.mjs` covers all three schema files in `api/` (verified against the file list). Model ids (`claude-opus-5`, `claude-opus-4-8`) are current.

### 2d. Secrets & Credential Exposure

**[config] - `.env` is tracked in git despite the ignore rule**
Severity: 🟡 Medium
Location: .env (tracked since 7f29390)
Description: `.gitignore` excludes `.env` but the file was committed before the rule was added and has stayed tracked. Every revision was inspected: it has only ever held the Supabase project URL, the `sb_publishable_` anon key, an empty `VITE_STRIPE_AUTHORIZE_URL` and an empty `ANTHROPIC_API_KEY`. Nothing secret has leaked, so no rotation is needed. The risk is the next edit: anyone who pastes a real `ANTHROPIC_API_KEY` into the file they were told to use for local development commits it. A side effect worth knowing: the production serverless environment reports `VITE_SUPABASE_ANON_KEY: MISSING` (live `/api/stripe/status`), yet the live bundle contains the publishable key - the committed `.env` is what Vite's `loadEnv` reads at build time, so production's browser configuration is coming from this file, not from Vercel.
Verified by: `git ls-files -ci --exclude-standard`; `git log -p -- .env` for every revision; live `/api/stripe/status` env block; grep of the live `index-*.js` for `sb_publishable_`.
Recommendation: `git rm --cached .env`, set `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` in the Vercel project for all environments, redeploy, and confirm the bundle still carries the key. Add a pre-commit secret scan (Section 7b).
Status: New

**[config] - Production has a Supabase secret key in `SUPABASE_ANON_KEY`**
Severity: 🟠 High
Location: Vercel production environment (reported by api/stripe/status.ts:90-93)
Description: The deployed `/api/stripe/status` returns `checkoutReady: false` with the single blocking item "SUPABASE_ANON_KEY holds a secret key". `api/_lib/stripe.ts:anonKey()` feeds that value to every `createClient()` used for `loadCatalogue()`, `userFromRequest()` and `isAdminRequest()`, and the `adminGate()` copies in `conceive.ts` and `marketing.ts`. Where a user bearer token is also set, PostgREST authorises on the bearer and the damage is contained; where it is not (`loadCatalogue`), the read runs as `service_role`. I found no path through which a request can turn this into data it should not see, which is why this is High rather than Critical, but a secret key is in a slot the code treats as public-grade, three handlers share the mistake, and the site's own health check has been reporting "not ready" in production without anyone acting on it.
Verified by: Live GET of `https://www.maisonobsidian.com.au/api/stripe/status` (a request any visitor can make).
Recommendation: Replace `SUPABASE_ANON_KEY` in Vercel with the publishable/anon key, redeploy, confirm `checkoutReady: true`. Treat the secret key that was in the slot as exposed to every log line that ever printed a client config and rotate it in the Supabase dashboard.
Status: New

**[database] - A default staff passphrase was committed in history (superseded)**
Severity: 🟢 Low
Location: commit 85e7a1c (original 0025_staff_desk.sql seed row)
Description: The first version of `0025` seeded `staff_access` with a fixed passphrase (`crypt('obsidian-change-me', ...)`). The current file seeds a random UUID and `0029` makes `staff_ok()` ignore stored hashes entirely, so the value has no effect on any database where `0029` has been applied (it has: `staff_members` exists in production). Recorded so a future reader does not treat it as live.
Verified by: `git log -p -S "gen_salt('bf')"`; live PostgREST 401 on `staff_members` confirms `0029` is applied.
Recommendation: None beyond confirming `0029` is applied on every environment, which Section 7 covers.
Status: New

No Stripe secret, webhook secret, Anthropic key, Mailgun/Resend key or JWT-shaped token appears anywhere in git history (`git log -p --all -S` for each prefix; the only `sb_secret_` hits are the string comparisons in `status.ts`). No `VITE_`-prefixed variable carries a secret. The service-role key is read only in `api/_lib/stripe.ts:serviceClient()` and never logged.

### 2e. API Security Surface

**[config] - No security headers on the deployed site or API**
Severity: 🟠 High
Location: vercel.json (no `headers` block); verified on https://www.maisonobsidian.com.au/ and /api/*
Description: The live HTML and every API response carry `strict-transport-security: max-age=63072000` and nothing else: no `Content-Security-Policy`, no `X-Frame-Options` or `frame-ancestors`, no `X-Content-Type-Options`, no `Referrer-Policy`, no `Permissions-Policy`. The `www` host's HSTS lacks `includeSubDomains` and `preload` (the `.vercel.app` alias has both). The Supabase session is in `localStorage` (supabase-js default, `src/lib/supabase.ts:9`), so a single injected script anywhere on the origin takes the customer's session. The storefront loads Google Fonts and gtag.js and renders user-influenced strings (reviews, chat replies, engraving) - all through React, which is why there is no XSS finding in 2b, but a CSP is the layer that holds when the next contributor reaches for `dangerouslySetInnerHTML`. The site can also be framed by any origin.
Verified by: `curl -sI` on the apex (308), `www` (200), a product page, `/api/stripe/status`, `/api/chat` and a static asset. Headers recorded as returned.
Recommendation: Add a `headers` block to `vercel.json` for `/(.*)`: `Content-Security-Policy` (`default-src 'self'; script-src 'self' https://www.googletagmanager.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; font-src https://fonts.gstatic.com; img-src 'self' data: blob: https://*.supabase.co; connect-src 'self' https://*.supabase.co https://www.google-analytics.com https://*.googletagmanager.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self' https://checkout.stripe.com`, tightened after checking the console), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy: camera=(), microphone=(), geolocation=()`, and HSTS with `includeSubDomains; preload`. Inline style objects are used throughout, so `style-src` needs `'unsafe-inline'` for now; move to a nonce later. Verify with `curl -sI` after deploy.
Status: New

**[api] - Checkout and shipping-quote routes have no rate limiting and each call reaches Australia Post and Stripe**
Severity: 🟠 High
Location: api/shipping/quote.ts:15-47; api/stripe/checkout.ts:17-179
Description: Both routes are unauthenticated and uncounted. Each `quote` call makes one Australia Post PAC request; each `checkout` call makes a PAC request, a Stripe customer lookup/create for signed-in users and a Checkout Session create. The PAC API key has a request quota, and exhausting it does not degrade gracefully: `quoteRates()` throws, the bag shows "Postage quotes are unavailable" and the postal checkout path is blocked for every real customer until the quota resets. A loop over `/api/shipping/quote` with a valid bag is all it takes. Stripe-side, unbounded `checkout.sessions.create` calls fill the dashboard with abandoned sessions and, with `after_expiration.recovery` enabled, mint recovery links.
Verified by: Read only - not load-tested against production.
Recommendation: Apply the same shared-store rate limiter recommended in 2c to both routes (per IP per minute, with a stricter cap on `checkout`), cache PAC results for a (bag-shape, postcode) key for an hour, and add a Cloudflare or Vercel WAF rule in front of `/api/` if the plan allows. Surface a PAC quota alert.
Status: New

**[api] - `/api/stripe/status` discloses deployment configuration to anyone**
Severity: 🟡 Medium
Location: api/stripe/status.ts:39-103
Description: The route is documented as deliberately unauthenticated ("never returns key material, so it is safe to call without auth"). It is true that no key value is returned. What is returned today, live: that Stripe is in **live mode** (`sk_live…` prefix), that the webhook secret is set, the `SITE_URL`, which Supabase key kinds sit in which slots (including the current misconfiguration), that Australia Post is configured, and `AUSPOST_FROM_POSTCODE: 4179` - the suburb the business posts from. That is reconnaissance an attacker would otherwise have to work for, served with `Cache-Control: no-store` on every request. The documented position ("safe to call without auth") covers key material; it does not cover the posting postcode or the config-state oracle.
Verified by: Live GET as an anonymous visitor.
Recommendation: Gate the route behind `isAdminRequest()` (return 404 otherwise), or at minimum drop the postcode and key-kind details for unauthenticated callers and keep only `checkoutReady`. Keep the full report for admins; it is useful.
Status: New

**[api] - Stripe return URLs derived from request headers when `SITE_URL` is unset**
Severity: 🟢 Low
Location: api/_lib/stripe.ts:76-82 (`siteUrl`)
Description: `siteUrl()` falls back to `x-forwarded-proto` / `x-forwarded-host` / `host` to build `success_url`, `cancel_url`, the billing-portal `return_url` and the waitlist email link. `.env.example` documents the fallback as intended ("right for every deployment including previews"). On Vercel the forwarded host is set by the platform, which bounds the risk to "a preview alias could be the return host", and production has `SITE_URL` set (to the apex, which 308-redirects every return to `www`; see Section 8b). The workspace's own `check-request-origin` gate exists because this pattern has shipped a `0.0.0.0` origin before. Recorded as Low because the bound holds today.
Verified by: Live `/api/stripe/status` shows `SITE_URL` set.
Recommendation: Set `SITE_URL` to `https://www.maisonobsidian.com.au` and make the fallback a hard failure in production (`VERCEL_ENV === "production" && !SITE_URL` → 500), keeping the header fallback for previews only.
Status: New

CORS is correct by omission: no API response sets `Access-Control-Allow-Origin`, so browsers enforce same-origin; the `*` on static files is Vercel's default for assets and carries no credentials. HTTP methods are enforced on every route (405 verified live on GET to the POST routes). Error responses are generic for non-admins (`route()` wrapper).

### 2f. Payments Integrity (Stripe)

The core is sound and was exercised as far as the rules allow:

- **Price authority:** `priceLines()` ignores every client price; `unitPrice`/`label` from the bag are validated (`label` may only be "Discovery Box", boxes must be complete sets of five, quantity 1-20, lines ≤ 100). `npm run test:security` passes and asserts hostile quantities, a fake bundle label and partial boxes are refused. Postage is re-quoted server-side and the chosen service must exist in the fresh quote.
- **Webhook:** `config.api.bodyParser = false`, raw bytes read before `req.body`, `constructEvent` with the secret, 400 on bad or missing signature, `payment_status === "paid"` checked before recording. Replays are safe: `record_paid_order` takes `pg_advisory_xact_lock(hashtextextended(session_id))` and records a `processed_checkout_sessions` marker; subscription starts and renewals are keyed on unique `stripe_subscription_id` / `invoice_id`.
- **State changes** only from verified webhook events or a server-side `stripe.checkout.sessions.retrieve` in `confirm.ts`; `confirm` refuses a session whose `metadata.user_id` is not the caller.
- **One active subscription** is enforced by a partial unique index (`0032`) and by a Stripe-side check before creating the session; a second paid one is cancelled and refunded with an idempotency key.

Findings:

**[api] - The README's authorise-later batch model is not what runs; no manual-capture path exists**
Severity: 🟡 Medium
Location: README.md:6-9, 317-323, 491-510; src/lib/stripe.ts:1-57; api/stripe/checkout.ts:153-158
Description: The README's opening paragraph, "The batch model" and "Payments (Stripe, authorize-later)" sections all describe cards being held and captured when a batch is met, and a `capture-batch` Edge Function. `checkout.ts` creates an ordinary `mode: "payment"` session with no `capture_method: "manual"`; `STRIPE_INTEGRATION_TODO.md` says so ("Payment is taken at checkout, like any normal store"); `supabase/functions/` has no `capture-batch`. `src/lib/stripe.ts:authorizePayment()` still exists and mints `pi_stub_*` ids, and the only remaining caller is the admin "Bill month N" button (next finding). The question in the review brief - what happens when a 7-day authorisation expires before a batch is met - has the answer "nothing, because nothing is authorised", but a reader of the README would not know that, and `STRIPE_INTEGRATION_TODO.md` contradicts the README on `allow_promotion_codes` (TODO says `false`, code says `true`) and on the API version (TODO says none pinned, `api/_lib/stripe.ts:21` pins `2026-08-26.dahlia`).
Verified by: Read of both documents against the handlers.
Recommendation: Rewrite the README's payment sections to describe immediate charge via hosted Checkout, delete `authorizePayment()` and `VITE_STRIPE_AUTHORIZE_URL`, and correct the two TODO rows. If a batch/pre-order model is still wanted, it is a feature to design, not a documentation fix.
Status: New

**[frontend] - Admin "Bill month N" button calls a revoked RPC with a stub payment id and reports nothing**
Severity: 🟡 Medium
Location: src/components/AdminSubscriptions.tsx:50-59 and 132-136; src/lib/subscription.ts:305-324
Description: For any subscription without a `stripe_subscription_id` the console offers "Bill month N". It calls `authorizePayment()` (returns `pi_stub_…` since no endpoint is configured) then `bill_subscription_month`, which `0029` revoked from `authenticated`. The RPC fails, `billSubscriptionMonth` returns `false`, the component ignores the return value and reloads. No subscription without a Stripe id can exist any more (`start_subscription` is revoked too), so the button is dead, but it is dead in a way that looks like it worked.
Verified by: Read only.
Recommendation: Remove the button, `authorizePayment`, `billSubscriptionMonth` and `startSubscription` from the client. Surface RPC errors in the admin UI generally (most `admin.ts` helpers return `!error` and the callers discard it).
Status: New

**[config] - A Stripe test publishable key is compiled into the production bundle while the server is in live mode**
Severity: 🟡 Medium
Location: src/lib/social.ts:21 (`socialProfiles(import.meta.env)` forces Vite to inline the whole env object); Vercel production env (`VITE_STRIPE_PUBLISHABLE_KEY`)
Description: Because `socialProfiles()` receives `import.meta.env` as an object, Vite inlines **every** `VITE_*` variable into `index-*.js`. The live bundle therefore contains `VITE_STRIPE_PUBLISHABLE_KEY` holding a `pk_test_` key (nothing in the code reads it - hosted Checkout needs no publishable key), plus Vercel's own `VITE_VERCEL_*` metadata (deployment id, branch URL, git author login). A publishable key is public by design, so this is not a leak; it is a test-mode artefact in a live-mode deployment and a mechanism by which the next `VITE_`-prefixed secret ships to every visitor without anyone adding a reference to it.
Verified by: Grep of the live `index-Cn2WAj3Y.js` for the inlined env object.
Recommendation: Change `socialProfiles()` to read the four named keys explicitly (`import.meta.env.VITE_INSTAGRAM_URL` etc.) and pass a plain object from the prerender script; delete `VITE_STRIPE_PUBLISHABLE_KEY` from Vercel; add `envPrefix` discipline to the README.
Status: New

**[frontend] - A failed confirmation on the thank-you page is reported as "Order placed"**
Severity: 🟢 Low
Location: src/App.tsx:298-302
Description: When `confirmStripeSession()` returns `null` (404, non-JSON, network failure) the branch comments "No backend to ask (the offline demo): the payment stands on its own", clears the bag and shows the paid state. In production the customer only lands on `/thanks?session_id=` after Stripe redirects them, so the payment did happen and the webhook will record it; the page is right in substance. It is wrong in the one case that matters - the webhook and the confirm route both failing - where the customer is told the order is placed and nobody has a row for it.
Verified by: Read only.
Recommendation: In configured mode, treat a null confirm as `status: "error"` with the existing "if your card was charged the order is safe" copy, and keep the bag until a confirm succeeds.
Status: New

`npm run test:security` **passes** (run from a clean `npm ci`). It compiles the real `api/` modules and asserts price isolation, quantity bounds, bundle validation, multi-chunk metadata round-trips, paid-only recording and RPC error propagation. It does not touch the webhook signature path, `recordSubscriptionStart` or the migration itself (Section 4c).

### 2g. Dependency Security

**[config] - 14 dev-tree advisories, all with fixes available**
Severity: 🟢 Low
Location: package-lock.json (`vite 7.3.1`, `postcss`, `picomatch`, `source-map-js`, `launch-editor`)
Description: `npm audit --omit=dev --audit-level=moderate`: **0 vulnerabilities** in the production tree. `npm audit --audit-level=moderate`: 14 (10 high, 2 moderate, 2 low), all in the Vite/PostCSS build chain: dev-server arbitrary file read and `server.fs.deny` bypasses (GHSA-4w7w-66w2-5vf9, GHSA-v2wj-q39q-566r, GHSA-p9ff-h696-f583), PostCSS source-map disclosure, picomatch ReDoS. These matter on a developer machine running `npm run dev` on a network interface, not in the deployed bundle.
Verified by: `npm audit` output after `npm ci`.
Recommendation: `npm audit fix` (non-breaking), commit the lockfile, rebuild. Enable Dependabot (Section 6).
Status: New

Devdependencies do not reach the serverless functions: `api/` imports only `stripe`, `@supabase/supabase-js` and `@anthropic-ai/sdk`, all in `dependencies`. The Edge Function's unpinned `esm.sh` import is covered under 2a.

### 2h. Supabase Row-Level Security & Database Functions

Every table in `public` has RLS enabled (checked table by table across all 35 migrations). The final-state policy set is: `fragrances` public read; `commits`/`shipments`/`chat_messages`/`scent_requests`/`scent_subscriptions`/`subscription_deliveries`/`customer_profiles`/`scent_profiles`/`scent_signals`/`stripe_customers` read-own plus admin-read; `reviews` published-only read with a column grant that hides `user_id`; `marketing_signups`/`waitlist` admin-read; `order_fulfilment`/`staff_access`/`staff_members`/`processed_checkout_sessions` no policies (RPC-only). Every `SECURITY DEFINER` function pins `search_path`. The `marketing_audience` view sets `security_invoker = true`. Storage policies restrict `fragrance-images` writes to `is_admin()`.

**Verified live, as `anon`, against the production project (reads only):** `commits`, `subscribers`, `scent_profiles`, `order_fulfilment`, `stripe_customers`, `customer_profiles`, `marketing_signups` all return `[]`; `waitlist` and `staff_members` return `401 permission denied` (the `0029`/`0035` grants); `reviews?select=user_id` returns `401 permission denied for table reviews` (the column grant holds). I did not test as a second ordinary user and did not attempt any write.

**[database] - Inventory, raw-oil and threshold columns are readable by every visitor**
Severity: 🟢 Low
Location: supabase/migrations/0001_init.sql:196-198 (`fragrances_read using (true)`); src/lib/store.ts:61 (`select("*")`)
Description: The public-read policy exposes every column, so `stock_10ml/30ml/50ml`, `stock_car/wash/moist`, `oil_ml`, `low_stock_threshold`, `format_prices`, `moq`, `committed` and `vip_only` are readable by anyone with the anon key, and the storefront fetches them on every load. Live probe: `f14` returns `oil_ml: 600, stock_10ml: 24, stock_30ml: 12, stock_50ml: 6`. Stock drives the "Ready to ship" copy so some of it is shown anyway; raw oil on hand and the low-stock threshold are internal operating data. The `0033` migration shows the pattern to use (`revoke all; grant select (cols)`).
Verified by: Live PostgREST GET as anon.
Recommendation: Column grants on `fragrances` for `anon`/`authenticated` that exclude `oil_ml` and `low_stock_threshold`; have `store.ts` and `prerender.mjs` select named columns. Admin reads of the hidden columns go through an RPC or a service-role path.
Status: New

**[database] - Unauthenticated write RPCs have no rate limit or size bound**
Severity: 🟢 Low
Location: 0005 `log_chat_message` (8,000 chars/row, anon), 0010 `request_scent`, 0019/0020 `save_scentprint` (unbounded `jsonb`), 0024 `record_scent_signal`, 0014 `join_inner_circle`, 0035 `join_waitlist`
Description: Each is granted to `anon` and inserts one row per call with at most a length cap on text fields; `save_scentprint` accepts any JSON object as `dims`. Nothing bounds rows per IP, so the tables grow at the rate an attacker chooses. Supabase's gateway applies a coarse global rate limit; nothing here is per-caller.
Verified by: Read only.
Recommendation: Cap `jsonb` payload size and key set in `save_scentprint`/`record_scent_signal` (validate keys against the sixteen dimensions, values 0-100), and add retention (Section 2i).
Status: New

**[database] - Several seed and data migrations are not safe to re-run, and demo inventory is live in production**
Severity: 🟡 Medium
Location: 0002_seed.sql:40-57; 0003_admin_inventory.sql:35-37; 0006_oil_inventory.sql:12; 0021_audience_catalogue.sql:58-82; 0027_house_prices.sql; 0028_house_price_30ml.sql; docs/SUPABASE_SETUP.md:75-79
Description: `0003` sets `stock = 24/12/6` on any row whose stock is all zero - which on a production database is every sold-out fragrance. `0006` sets `oil_ml = 600` on any row at zero. `0027`/`0028` set every fragrance's three prices and strip per-format overrides unconditionally, reverting any price an admin has set since. `0002` and `0021` `on conflict do update` every descriptive field, reverting admin edits to copy, prices and VIP flags, and `0021` deletes four fragrances. The setup guide says "every migration except `0002_seed.sql` is safe to re-run". It is not. Separately, the live database still carries the `0003`/`0006` demo numbers (`f14`: stock 24/12/6, oil 600 - exactly the seed values), so the storefront's "Ready to ship" availability note (`BagDrawer.tsx:availabilityNote`) and the merchant feed's `in_stock` are being driven by demo data for any fragrance the admin has not touched.
Verified by: Migrations read in order; live PostgREST read of `f14`.
Recommendation: Guard every seed/demo `update` with an environment check or move demo stock/oil into a separate `seed_demo.sql` that is never applied to production; make the price migrations one-shot (a `migrations_applied` check or a comment that they must not be re-run); correct the setup guide; and have the admin set real stock and oil figures (or zero them) so availability copy is true.
Status: New

**[database] - Batch counters cannot be inflated by clients at HEAD**
Severity: - (informational)
Description: After `0029`, `commits` accepts no client writes and `commit_to_batch` is revoked, so the `committed` counter (trigger-maintained) and stock columns (admin RPCs with `greatest(0, …)`) have no anonymous path. `record_paid_order` is serialised per session by an advisory lock. The read-then-write in `bill_subscription_month` uses `for update`. No finding.

### 2i. Session & Data Handling

**[frontend] - No privacy policy, returns policy or support contact is published on the live site**
Severity: 🟠 High
Location: src/components/Help.tsx:4-21; Vercel production env (`VITE_SUPPORT_EMAIL`, `VITE_PRIVACY_POLICY_URL`, `VITE_RETURNS_POLICY_URL` unset); src/components/BagDrawer.tsx:153 and 159; src/components/Checkout.tsx:491-492; api/chat.ts:26; src/lib/concierge.ts:136
Description: The live bundle's inlined env object contains none of the three variables, so `/help` renders "A detailed privacy policy is not yet published here. Please confirm how your information will be handled before submitting personal details", "A detailed returns policy is not yet published here" and "Online support contact details are not yet available here". Meanwhile the site collects email, name, street address and mobile at checkout, stores concierge transcripts for anonymous visitors indefinitely (`0005`), sends customer photographs to Anthropic (`ScentMemory`), builds taste profiles, and takes live card payments. The Australian Privacy Act's APP 1 (a published policy) and APP 5 (notice at collection) apply from the first customer; the Scent DNA lead form and sign-up tick boxes describe consent well but link to nothing. On the consumer-law side the bag, checkout and concierge all promise "30-day returns" and "free shipping over $100" against no published terms. `docs/SECURITY_SHOPPER_FLOW_ROLLOUT.md` step 5 lists these as pre-launch items; the site launched without them.
Verified by: Grep of the live `index-*.js` inlined env object; live `/help` snapshot.
Recommendation: Publish a privacy policy (covering Supabase, Stripe, Resend, Google Analytics, Anthropic as processors; photographs; transcripts; retention; access/deletion requests) and a returns/refunds policy consistent with ACL guarantees and the "30-day" promise; set the three variables in Vercel; link the privacy policy from the checkout, the footer sign-up, the Scent DNA lead form and the Scent Memory upload; add a support email. Until then, remove the "30-day returns" line.
Status: New

**[database] - Single opt-in marketing and waitlist capture with no address verification**
Severity: 🟡 Medium
Location: supabase/migrations/0014_profiles_marketing.sql:58-79 (`join_inner_circle`); 0035_waitlist.sql:42-69 (`join_waitlist`); 0019/0020 (`attach_scentprint_email`); src/components/Footer.tsx:49-54
Description: Anyone can type any address into the footer box, a Notify-me form or the Scent DNA lead form, and that address is recorded as having consented (`opted_in = true`, "Express consent, recorded with the time and source"). No confirmation email is sent, so the recorded consent is the typist's, not the address holder's. Under the Spam Act 2003 consent must come from the account holder; a single-opt-in list also fills with typos and spite sign-ups, which hurts Resend deliverability. The same forms let a stranger sign a third party up.
Verified by: Read only.
Recommendation: Double opt-in: send a confirmation via Resend with a signed token and only set `opted_in = true` when it is followed; store `confirmed_at`. For the waitlist, the "one email" model can stay single opt-in if the launch email itself carries an unsubscribe link (today it has none; `waitlist.ts:60` says "the only one we'll send" instead).
Status: New

**[database] - No retention or deletion path for customer data**
Severity: 🟢 Low
Location: 0005_chat.sql; 0019_scent_dna.sql; 0024_scent_signals.sql; 0014_profiles_marketing.sql
Description: Anonymous chat transcripts, Scentprints with emails, signals with free-text notes and marketing signups accumulate with no TTL, and there is no account-deletion flow in the app or a documented operator procedure. Customers can withdraw marketing consent (`set_my_consents`) but cannot have their data erased.
Verified by: Read only.
Recommendation: A `pg_cron` job to delete `chat_messages` with `user_id is null` after 30 days and signals/profiles with no `user_id` and no email after 12 months; a documented deletion runbook (Supabase auth delete cascades `customer_profiles`, `scent_subscriptions`, `reviews`; `commits.user_id` sets null and must be scrubbed of PII separately).
Status: New

Positives: sensitive values are never logged in `api/` (the only `console.error` payloads are error objects and event types); the Stripe session id is dropped from the URL and excluded from GA page views; GA sends no PII and only item ids/prices; the bag, orders and Scentprint in `localStorage` are non-sensitive by design; CSRF is not applicable (no cookie auth; bearer tokens only).

---

## Section 3: Performance

### 3a. Database Performance

**[database] - `select('*')` on the catalogue for every visitor**
Severity: 🟢 Low
Location: src/lib/store.ts:61; scripts/prerender.mjs:120
Description: Every page load pulls all ~30 columns for 59 rows including the inventory columns discussed in 2h. Small today; the fix is the same column grant.
Verified by: Read only.
Recommendation: Select the columns the storefront renders.
Status: New

**[database] - Missing index on `commits.payment_intent_id`**
Severity: 🟢 Low
Location: api/_lib/record.ts:223 and 231 (`.eq("payment_intent_id", pi)`); supabase/migrations/0001_init.sql:59-61
Description: Refund and dispute handling update `commits` by `payment_intent_id`, which has no index (there are indexes on `fragrance_id`, `user_id`, `status`, `checkout_session_id` and `lower(user_email)`). Full scan on every refund webhook.
Verified by: Read only.
Recommendation: `create index if not exists commits_payment_intent_idx on public.commits(payment_intent_id);`
Status: New

Unbounded reads are confined to admin surfaces (`fetchAllCommits`, admin subscriptions; `useWaitlist` caps at 5,000, reviews at 100/200). No N+1 patterns in the API; `recordRenewal` does three sequential reads that could be one, trivial at current volume. The `committed` trigger is O(1) per row. No Realtime subscriptions are used.

### 3b. Serverless / Backend Performance

**[api] - No timeouts or `maxDuration` on the Claude routes**
Severity: 🟢 Low
Location: api/chat.ts:15; api/scent-ai.ts:23; api/conceive.ts:17 (only api/marketing.ts:13 sets `maxDuration: 60`)
Description: `new Anthropic({ apiKey })` uses the SDK default timeout (10 minutes) and the routes set no `maxDuration`, so a slow upstream holds the function for the platform maximum. `scent-ai` with `max_tokens: 8192` on Opus can legitimately run long.
Verified by: Read only.
Recommendation: `new Anthropic({ apiKey, timeout: 45_000, maxRetries: 1 })` and `export const config = { runtime: "nodejs", maxDuration: 60 }` on each.
Status: New

Prompt caching is real: `scent-ai.ts:cacheableSystem()` places `cache_control: ephemeral` on the catalogue block as the README claims. Heavy SDKs are imported only by the routes that use them. Independent awaits are parallel where it matters (`prerender.mjs:179`). `Cache-Control: no-store` is set on every API response, which is correct for personalised ones and slightly wasteful for `/api/shipping/quote` (addressed by the cache recommendation in 2e).

### 3c. Frontend Performance

**[frontend] - Main bundle weight and a PNG hero**
Severity: 🟢 Low
Location: vite.config.ts:28-35; public/assets/bottle-pair.png; src/components/BottleImage.tsx
Description: Production build: `index` 321 KB (88 KB gz), `supabase` 211 KB (55 KB gz), `react` 192 KB (60 KB gz), CSS 26 KB; admin (64 KB), staff desk (18 KB) and Scent DNA (123 KB) are lazy chunks, which is the right split. The live homepage HTML references 736 KB of script/CSS before any image. The hero/OG image `bottle-pair.png` is a 314 KB PNG at 1024×559 that would be ~40 KB as WebP; `public/assets` is 25 MB and ~36 of its 101 files are unreferenced spares (including a 6.8 MB `oud wood.png`) that are copied into `dist/` (30 MB) and deployed on every build. `BottleImage` sets `loading="lazy"` but no intrinsic `width`/`height`, so tiles shift as images arrive. Fonts: three families from Google Fonts with `display=swap` and preconnect - fine.
Verified by: `npm run build` output; `curl -sI` on the live assets; `du`/`sips` on `public/assets`.
Recommendation: Convert `bottle-pair.png` and the per-slug PNGs to WebP (`trim_bottle_renders.py` is the natural home), delete or move unreferenced spares out of `public/`, add `width`/`height` to `BottleImage`, and consider loading the Supabase client lazily on pages that never read live data (the prerendered content already covers first paint).
Status: New

Prerendering works: `dist/fragrance/<slug>/index.html` carries per-page title, description, canonical, OG tags, JSON-LD and a readable snapshot in `#root`; `vercel.json` rewrites only paths that are not files to `app.html`, so static pages win. Verified live on `/fragrance/smoky-obsidian/`.

---

## Section 4: Code Health & Cleanliness

### 4a. Code Quality

**[frontend] - Legacy payment and billing code survives `0029` and silently swallows failures**
Severity: 🟡 Medium
Location: src/lib/store.ts:89-112 (`recordCommit`, uncalled); src/lib/stripe.ts:1-57 (`authorizePayment`); src/lib/subscription.ts:201-242 (`startSubscription`, uncalled) and 305-324; src/components/AdminSubscriptions.tsx:50-59; .env.example `VITE_STRIPE_AUTHORIZE_URL`
Description: `recordCommit` ("Errors are swallowed so a backend hiccup never blocks the optimistic UI") and `startSubscription` have no callers; `authorizePayment` and `billSubscriptionMonth` have one dead caller (2f). All four target RPCs that `0029` revoked. The pattern - `return !error` with the caller discarding it - recurs across `admin.ts` (`adminSetOil`, `adminDeleteFragrance`, `adminSetStock`, `adminSetFormats`, `adminCreateShipment`) so an admin action that is refused by the database looks identical to one that worked.
Verified by: grep for call sites; read of each function.
Recommendation: Delete the dead paths; return the error message from every admin helper and show it (the `adminUpsertFragrance` → `saveError` pattern already exists).
Status: New

**[api] - Pricing rules duplicated between `src/lib/formats.ts` and `api/_lib/catalogue.ts` with no test holding them together**
Severity: 🟡 Medium
Location: api/_lib/catalogue.ts:1-5 (comment: "the route test compares them against the seed catalogue"); scripts/test_checkout_security.mjs:27-39
Description: Format definitions, `RITUAL_DISCOUNT`, `SUBSCRIPTION_DISCOUNT`, `DISCOVERY_BOX_PRICE`, default prices and the `buyable`/`formatStatus` logic exist twice, once per runtime, by design (the API cannot import Vite-resolved modules). They agree today (checked constant by constant). The comment says the route test compares the two; it does not - the test builds a synthetic fragrance and never imports `src/lib/formats.ts`. A price change in one file silently diverges the displayed price from the charged price.
Verified by: Read of both files and the test.
Recommendation: Add a test that imports both modules (the test already transpiles `api/`; `formats.ts` transpiles the same way with `HOUSE_PRICE` stubbed) and asserts `formatPrice`, `subscriptionPrice` and `buyable` agree across the seed catalogue for every format. Or move the shared rules to a `.js` module both can import.
Status: New

**[frontend] - Catalogue falls back to seed data silently in production**
Severity: 🟡 Medium
Location: src/lib/store.ts:57-69 and 75; README.md:641-643
Description: The README documents this as deliberate ("if the vars are absent or a request fails, it silently falls back to the seed catalogue so the app never breaks"). That was right for a demo. In production the seed (`data.ts`, 56 entries) and the live catalogue (59 rows, with admin-added scents, launch dates, real images and current prices) are different data sets: a failed read now renders retired or unlaunched scents, demo stock and stale prices with no indication, and `loaded` stays true so a missing slug 404s. Checkout prices server-side, so money is safe; the storefront is not truthful.
Verified by: Read only.
Recommendation: In configured mode, treat a failed catalogue read as an error state (retry, then a "catalogue temporarily unavailable" banner) rather than seed; keep the seed fallback for `!supabase` only.
Status: New

**[frontend] - 33 `any` casts and a `strict: false` type-check**
Severity: 🟢 Low
Location: tsconfig.json:16 (`strict: false`); grep count of `: any|as any|@ts-ignore|@ts-expect-error` across src/ and api/
Description: `api/` handlers type `req`/`res` as `any` throughout; `src/` is type-checked under the non-strict config (Section 7b). The strict config fails on three `supabase` null checks in `staffDesk.ts:119-139`.
Verified by: `npx tsc -p tsconfig.app.json --noEmit` (exit 2).
Recommendation: Covered by the 7b finding.
Status: New

No commented-out code blocks and no `TODO`/`FIXME` markers exist in `api`, `src`, `supabase` or `scripts` (grep empty).

### 4b. Architecture Consistency

The split is coherent: trusted logic (pricing, postage, order recording, admin checks) lives in `api/` and the RPCs; the client holds UI state and demo stores. Routing is path-based with a legacy-hash migration, and the prerender agrees with `vercel.json`. Styling is inline style objects plus four CSS files, consistent with the README. The weak point is the demo/production boundary: `supabase === null` is the only switch, and the fallbacks in 4a (catalogue, confirm, cancel-subscription) let production drift into demo behaviour on transient failure rather than failing visibly.

### 4c. Testing

**[scripts] - Three gates, one assertion suite, no coverage of the webhook, migrations or RLS**
Severity: 🟡 Medium
Location: scripts/test_checkout_security.mjs; scripts/check_output_schemas.mjs; scripts/check_function_count.mjs
Description: All three run and pass (`test:security` PASS; schemas "all within the supported subset"; `api/: 12 of 12 serverless functions`). `check_function_count` and `check_output_schemas` are wired into `npm run build`; `test:security` is not run by anything automatic. Untested: webhook signature rejection (missing/forged `stripe-signature`), `recordSubscriptionStart`'s duplicate path, `prepareRenewal`/`recordRenewal`, the Resend senders, every RLS policy and every `SECURITY DEFINER` function, the migrations applied in order to a clean database (the README asserts this was done "against PostgreSQL 16" but nothing in the tree does it), the `0029` revocations, and the client/API pricing parity (4a). The function-count gate is at its ceiling (12 of 12), so the next route must be folded into an existing one, which is how `api/marketing.ts` grew a `waitlist-notify` action.
Verified by: Each script run locally.
Recommendation: Add `test:security` to the `build` script or a CI job; add a webhook test that feeds `constructEvent` a forged signature and asserts 400; add a migration test (`supabase start` + apply + a handful of `set role anon` assertions) that runs in CI; see Section 7b for CI.
Status: New

### 4d. Configuration & Environment Management

**[config] - `.env.example`, the docs and the code disagree on which variables exist**
Severity: 🟡 Medium
Location: .env.example; docs/SUPABASE_SETUP.md:408-428; api/_lib/recovery.ts:10-14; api/_lib/waitlist.ts:8-10; api/chat.ts:105
Description: Read by code but documented nowhere: `RESEND_API_KEY`, `RECOVERY_EMAIL_FROM`, `RECOVERY_REPLY_TO` (the entire transactional-mail configuration), `ANTHROPIC_KEY` (an alias), `SUPABASE_URL`/`SUPABASE_ANON_KEY` (in the TODO file only). Documented but read by nothing: `MAILGUN_API_KEY`, `MAILGUN_DOMAIN`, `MAILGUN_API_BASE`, `MAILGUN_FROM`, `VITE_STRIPE_AUTHORIZE_URL`. An operator following the docs configures Mailgun, sees nothing fail, and never receives an abandoned-checkout or launch email, because `sendRecoveryEmail()` and `waitlistNotify()` **fail open** - `console.warn` and return - when the Resend variables are unset. Missing server-side variables elsewhere fail closed correctly (501 with the names listed).
Verified by: The env-read grep from the instructions versus `.env.example`.
Recommendation: Rewrite `.env.example` and the setup guide's reference table from the grep output; rename `RECOVERY_EMAIL_FROM` to something that says it is the site-wide sender; make the recovery/waitlist senders return a 501-style error that reaches the admin UI when unconfigured.
Status: New

### 4e. Logging & Observability

**[config] - No error tracking, alerting or health check; webhook failures would go unnoticed**
Severity: 🟡 Medium
Location: api/stripe/webhook.ts:85-88; api/stripe/status.ts (config echo only)
Description: Errors are logged with route name and event type via `console.error`, which lands in Vercel's function logs and nowhere else. There is no error tracker, no alert on webhook 500s, no Stripe dashboard alert configured in the docs, and no endpoint that checks the database or Stripe reachability (`/api/stripe/status` checks that variables are set, not that anything works). If `record_paid_order` started failing (a migration not applied, a renamed column), Stripe would retry for three days and then give up, and the first sign would be a customer asking where their order is. Admin actions, staff-desk access and payment state changes are not logged anywhere queryable; the Stripe event id is not stored with the order.
Verified by: Read only.
Recommendation: Add Sentry (or Vercel's log drains) with an alert on any `api/stripe/*` 5xx; enable Stripe's "failed webhook" email; store `event.id` on `processed_checkout_sessions`; add a `/api/health` that runs `select 1` through the service client and a Stripe `balance.retrieve`, and point an uptime monitor at it. GA4 is correctly gated on the measurement id and sends no PII; consent gating is not implemented, which the privacy policy (2i) will need to address.
Status: New

---

## Section 5: Documentation

**[docs] - README is materially out of date**
Severity: 🟡 Medium
Location: README.md
Description: Diffed against `git ls-files` and the handlers. Mismatches: (1) the batch/authorise-later model and `capture-batch` Edge Function (lines 6-9, 217-225, 313, 317-323, 491-510) - not implemented, function absent; (2) "The staff desk… opens with a shared passphrase… bcrypt-hashed in `staff_access`" (413-448) - replaced by `staff_members` in `0029`, `admin_set_staff_passphrase` revoked; (3) components `MyReservations.tsx` and `LayoutSwitch.tsx` (274-276) - do not exist, `MyOrders.tsx` does; (4) the architecture tree lists migrations only to 0024 and "three tables" (295-311, 327) - there are 35 migrations and 20 tables; (5) Mailgun throughout `docs/SUPABASE_SETUP.md` §5 and `.env.example` - the code uses Resend; (6) `supabase functions deploy capture-batch` in the setup guide (245) under a heading that says the function was removed; (7) `concierge.ts` and the chat prompt still say "batch commits" (`ChatWidget.tsx:19-21`); (8) `STRIPE_INTEGRATION_TODO.md` says `allow_promotion_codes: false` and "no apiVersion" - both wrong. The README is also the only place the 2026-07 "validated against PostgreSQL 16" claim lives; nothing reproducible backs it.
Verified by: Name-by-name diff against the tree.
Recommendation: Rewrite the README's payment, staff desk, architecture and migration sections from the code; move the Mailgun SMTP section to a "Supabase auth mail" note and add a Resend section; delete the `capture-batch` instructions; fix the two TODO rows. Record the staff-desk change and the immediate-charge model in the instructions file's Settled positions.
Status: New

API routes and RPCs are documented only in file-header comments, which are good (every `api/` file states method, auth and body shape; every migration explains its intent). There is no single route table. The migration order and "add a migration" process are documented in the setup guide. `docs/SECURITY_SHOPPER_FLOW_ROLLOUT.md` is current but reads as a branch note; it does not say which of its eight steps were completed, and this review found steps 5 (policies) and 7 (role testing) unevidenced.

---

## Section 6: Dependency & Supply Chain Health

**[config] - No automated dependency updates, Node version undeclared, Python deps undeclared**
Severity: 🟢 Low
Location: package.json; (no .github/, .nvmrc, requirements.txt)
Description: `npm ci` is clean and the lockfile is in sync. No Dependabot or Renovate. No `engines`/`.nvmrc` (local Node 26 vs. whatever Vercel is set to). All ranges are `^`, so fresh installs can drift from the tested set between lockfile updates, which is normal with a committed lockfile. Nothing is vendored. `scripts/*.py` need `openpyxl` and `pillow` with no manifest.
Verified by: File listing; `npm ci`.
Recommendation: Add `"engines": { "node": "22.x" }` (or whatever Vercel runs) and a matching `.nvmrc`; add `.github/dependabot.yml` for npm (weekly, grouped minor/patch); add `scripts/requirements.txt`.
Status: New

---

## Section 7: Operational & Deployment Readiness

**[config] - Deployment is manual, undocumented end to end, and migrations are applied by hand**
Severity: 🟡 Medium
Location: docs/SUPABASE_SETUP.md; STRIPE_INTEGRATION_TODO.md; supabase/migrations/0028_house_price_30ml.sql:8-11
Description: Vercel builds from `main` (verified: `VITE_VERCEL_GIT_COMMIT_REF: main` in the live bundle). Migrations are pasted into the SQL editor ("0027 has already been applied to the live database" - `0028` header), so there is no record in the database of which have run; `supabase db push` is offered as an alternative but the setup guide warns it may fail on naming. There is no documented rollback, no backup/PITR note, no restore rehearsal, and no runbooks for the predictable failures (webhook failing, Australia Post quota or outage, Anthropic outage, Resend bounce). Vercel limits are respected: 12 functions (at the ceiling), `nodejs` runtime, body sizes small.
Verified by: Read of docs; live bundle metadata.
Recommendation: Adopt `supabase migration` tracking (link the project, `supabase db push` from CI, record applied versions); document Supabase's backup tier and run one restore into a scratch project; write four short runbooks in `docs/`; note the function-count ceiling and the plan for the 13th route.
Status: New

---

## Section 7b: Project Gates & Toolchain Conformance

**[config] - The build's type-check is non-strict and never covers `api/`; the strict config is dead**
Severity: 🟡 Medium
Location: package.json:8 (`tsc -b`); tsconfig.json (`strict: false`, no `references`); tsconfig.app.json (strict, unreferenced); tsconfig.api.json (unreferenced); eslint.config.js:9 (`globalIgnores(['dist', 'supabase/functions', 'api'])`)
Description: `tsc -b` with a `tsconfig.json` that has no `references` builds that file alone: `src/` under `strict: false`. `tsconfig.app.json` (strict, `noUnusedLocals`) is never invoked and **fails** today with three errors in `staffDesk.ts`. `tsconfig.api.json` is only run by hand (`docs/SECURITY_SHOPPER_FLOW_ROLLOUT.md` lists the command) - it passes - but neither the build nor Vercel runs it, and ESLint ignores `api/` and `supabase/functions` entirely. So the server code that handles money is neither linted nor type-checked by any automatic step.
Verified by: `npm run build` (passes), `npx tsc -p tsconfig.app.json --noEmit` (exit 2), `npx tsc -p tsconfig.api.json --noEmit` (passes), `npm run lint` (passes).
Recommendation: Make `tsconfig.json` a solution file with `references` to `tsconfig.app.json`, `tsconfig.api.json` and `tsconfig.node.json` (the Vite template shape), fix the three strict errors, remove `api` from the ESLint ignore list and add a Node-globals config block for it.
Status: New

**[config] - No CI, no pre-push hook, no secret scan; every gate runs only when a person remembers**
Severity: 🟡 Medium
Location: (no .github/workflows, no .husky, no hooks)
Description: `npm run build` runs the function-count and schema gates, and Vercel runs `npm run build`, so those two cannot be skipped on deploy. Everything else - `lint`, `test:security`, the API type-check, `npm audit` - is manual. There is no secret scan, which matters given a tracked `.env` and the AI-authored commit history (every non-merge commit in the log is authored "Claude"; a tool that writes keys into files needs a gate that catches them).
Verified by: Tree listing.
Recommendation: A single GitHub Actions workflow on push/PR: `npm ci`, `npm run lint`, `npx tsc -p tsconfig.api.json --noEmit`, `npm run test:security`, `npm run build`, `npm audit --omit=dev --audit-level=high`, plus gitleaks or the workspace's `check-no-secrets.sh`. Add the same as a `pre-push` hook. Enable Vercel's "require checks" so a red workflow blocks the deploy.
Status: New

---

## Section 8b: Additional Findings

### Accessibility

**[frontend] - Dialogs lack focus management**
Severity: 🟢 Low
Location: src/components/AuthModal.tsx:150-187; src/components/BagDrawer.tsx:69; src/components/ChatWidget.tsx:108-128; src/components/PasswordReset.tsx:59-80
Description: `role="dialog" aria-modal="true"` is set but no dialog moves focus into itself on open, traps Tab, or returns focus on close; `AuthModal` and `PasswordReset` do not close on Escape (`BagDrawer` does). Labels use `aria-label` rather than `aria-labelledby` on the visible heading. Form inputs across the site use `aria-label` consistently and `prefers-reduced-motion` is honoured; focus-visible styles exist for the Scent DNA and help surfaces but not for the gold CTAs generally. Contrast of `rgba(243,236,220,0.45)` microcopy on `#0b0b0d` is below 4.5:1 in several places (`micro` style at 0.35-0.5 alpha).
Verified by: Read of the components; not measured with a tool.
Recommendation: A small `useDialog` hook (initial focus, trap, Escape, restore) shared by the four dialogs; raise microcopy alpha to ≥0.6 or switch to `#9a927f`-class solid colours; run axe on the main flows.
Status: New

### Consumer law and product claims

**[frontend] - Third-party brand names in page titles, OG titles, structured data and image file names**
Severity: 🟢 Low
Location: src/lib/seo.ts:149-155 (`productTitle`: "… — Inspired by Tom Ford Black Lacquer | Maison Obsidian"); src/lib/snapshot.ts:37 (disclaimer, footer only); public/assets/ (24 files named for designer products, e.g. `Dior Sauvage.jpg`, `Creed Aventus Absolu.jpg`)
Description: Every product page's `<title>`, `og:title` and JSON-LD description names the reference house and fragrance. The merchant feed deliberately omits them (`feed.ts:7-10`) because "brand names in Shopping titles invite counterfeit-policy disapprovals", which shows the team has thought about it for Google and not for the pages. The snapshot footer carries an independence disclaimer; the rendered React page and the share card do not. The asset file names suggest the photography is styled after (or is) the designer products' own imagery. Comparative "inspired by" wording is lawful in Australia when it is not misleading; using a competitor's product photography is a different matter.
Verified by: Live product page `<head>`; file listing.
Recommendation: Get a short legal read on the title/OG pattern and on the provenance of the 24 brand-named photographs; surface the independence disclaimer on the product page itself. Confidence: Low - this is a legal question, not a code defect.
Status: New

### SEO and structured data

**[frontend] - Product `Offer` markup lacks `hasMerchantReturnPolicy` and `shippingDetails`**
Severity: 🟡 Medium
Location: src/lib/seo.ts:158-167 (`offerFor`)
Description: Live JSON-LD on `/fragrance/smoky-obsidian/` carries `ProductGroup` → `Product` variants with `image`, `sku`, `size` and an `Offer` with `price`, `priceCurrency`, `availability` and `itemCondition`. Search Console flags Offers without `hasMerchantReturnPolicy` and `shippingDetails` on every product; the workspace gates these fields for exactly that reason. `aggregateRating` is emitted only from real published reviews (`prerender.mjs:ratings()`), which is correct. Canonical, sitemap, robots and OG are correct and verified live; the sitemap's `lastmod` is the build date for every URL.
Verified by: Live `<script type="application/ld+json">` on a product page.
Recommendation: Add `shippingDetails` (`OfferShippingDetails` with `shippingDestination: AU`, free over $100, else a rate range) and `hasMerchantReturnPolicy` (30-day, once the policy exists per 2i) to `offerFor()`; drop `lastmod` or derive it from the fragrance's `updated_at`.
Status: New

**[config] - `SITE_URL` is the apex, which redirects every Stripe return**
Severity: 🟢 Low
Location: Vercel production env (`SITE_URL: https://maisonobsidian.com.au`); scripts/prerender.mjs:46-51 (canonical is `www`)
Description: The apex 308-redirects to `www` (verified). Stripe's `success_url` and the portal `return_url` therefore land on the apex and bounce; the query string survives, so it works, but every paying customer takes an extra hop and the two origins differ from the canonical the prerender uses.
Verified by: `curl -sI https://maisonobsidian.com.au/` (308 → www); live status shows `SITE_URL`.
Recommendation: Set `SITE_URL=https://www.maisonobsidian.com.au`.
Status: New

### Email deliverability

Resend requires a verified sending domain with SPF/DKIM; `RECOVERY_EMAIL_FROM` is undocumented and the status endpoint does not report whether it is set, so I cannot tell from the repository whether transactional mail is configured in production at all. The setup guide's Mailgun DNS section is detailed and correct for Mailgun, which is the wrong provider (4d). The recovery and waitlist emails carry "reply unsubscribe" text rather than a working unsubscribe link (2i).

---

## Section 9: Carried-Forward Items - Verification Results

None - this is the first review.

---

## Appendix: Finding Summary Table

| # | Area | Section | Severity | Title | Status |
|---|------|---------|----------|-------|--------|
| 1 | database | Security 2a | 🟠 High | Guest-order read policy trusts the JWT email claim without a verification check | New |
| 2 | edge-function | Security 2a | 🟠 High | `create-shipment` performs no authorisation and would buy labels for any caller | New |
| 3 | config | Security 2d | 🟠 High | Production has a Supabase secret key in `SUPABASE_ANON_KEY` | New |
| 4 | config | Security 2e | 🟠 High | No security headers on the deployed site or API | New |
| 5 | api | Security 2e | 🟠 High | Checkout and shipping-quote routes have no rate limiting and reach Australia Post and Stripe per call | New |
| 6 | frontend | Security 2i | 🟠 High | No privacy policy, returns policy or support contact published on the live site | New |
| 7 | database | Security 2a | 🟡 Medium | VIP-only not enforced on the server; VIP enrolment is free self-service | New |
| 8 | database | Security 2a | 🟡 Medium | `cancel_subscription` RPC callable directly, bypassing Stripe | New |
| 9 | api | Security 2c | 🟡 Medium | Public AI routes accept their system prompt from the request body | New |
| 10 | api | Security 2c | 🟡 Medium | Open LLM endpoints bounded only by a per-instance in-memory counter | New |
| 11 | config | Security 2d | 🟡 Medium | `.env` is tracked in git despite the ignore rule | New |
| 12 | api | Security 2e | 🟡 Medium | `/api/stripe/status` discloses deployment configuration to anyone | New |
| 13 | api | Security 2f | 🟡 Medium | README's authorise-later batch model is not what runs; no manual-capture path exists | New |
| 14 | frontend | Security 2f | 🟡 Medium | Admin "Bill month N" calls a revoked RPC with a stub payment id and reports nothing | New |
| 15 | config | Security 2f | 🟡 Medium | Stripe test publishable key compiled into the production bundle; whole env object inlined | New |
| 16 | database | Security 2h | 🟡 Medium | Seed/data migrations unsafe to re-run; demo inventory live in production | New |
| 17 | database | Security 2i | 🟡 Medium | Single opt-in marketing and waitlist capture with no address verification | New |
| 18 | frontend | Code Health 4a | 🟡 Medium | Legacy payment and billing code survives `0029` and swallows failures | New |
| 19 | api | Code Health 4a | 🟡 Medium | Pricing rules duplicated between client and API with no parity test | New |
| 20 | frontend | Code Health 4a | 🟡 Medium | Catalogue falls back to seed data silently in production | New |
| 21 | scripts | Code Health 4c | 🟡 Medium | Three gates, one assertion suite, no coverage of webhook, migrations or RLS | New |
| 22 | config | Code Health 4d | 🟡 Medium | `.env.example`, docs and code disagree on which variables exist; mail senders fail open | New |
| 23 | config | Code Health 4e | 🟡 Medium | No error tracking, alerting or health check | New |
| 24 | docs | Documentation 5 | 🟡 Medium | README is materially out of date | New |
| 25 | config | Operational 7 | 🟡 Medium | Deployment manual and undocumented; migrations applied by hand | New |
| 26 | config | Gates 7b | 🟡 Medium | Build type-check is non-strict and never covers `api/`; strict config is dead | New |
| 27 | config | Gates 7b | 🟡 Medium | No CI, no pre-push hook, no secret scan | New |
| 28 | frontend | Additional 8b | 🟡 Medium | Product `Offer` markup lacks `hasMerchantReturnPolicy` and `shippingDetails` | New |
| 29 | database | Security 2a | 🟢 Low | Earlier `SECURITY DEFINER` functions keep the default `PUBLIC` execute grant | New |
| 30 | api | Security 2b | 🟢 Low | Engraving and free-text fields have no character-set limit | New |
| 31 | api | Security 2c | 🟢 Low | Upstream error text returned to unauthenticated callers | New |
| 32 | database | Security 2d | 🟢 Low | A default staff passphrase was committed in history (superseded) | New |
| 33 | api | Security 2e | 🟢 Low | Stripe return URLs derived from request headers when `SITE_URL` unset | New |
| 34 | frontend | Security 2f | 🟢 Low | Failed confirmation on the thank-you page reported as "Order placed" | New |
| 35 | config | Security 2g | 🟢 Low | 14 dev-tree advisories, all with fixes available | New |
| 36 | database | Security 2h | 🟢 Low | Inventory, raw-oil and threshold columns readable by every visitor | New |
| 37 | database | Security 2h | 🟢 Low | Unauthenticated write RPCs have no rate limit or size bound | New |
| 38 | database | Security 2i | 🟢 Low | No retention or deletion path for customer data | New |
| 39 | database | Performance 3a | 🟢 Low | `select('*')` on the catalogue for every visitor | New |
| 40 | database | Performance 3a | 🟢 Low | Missing index on `commits.payment_intent_id` | New |
| 41 | api | Performance 3b | 🟢 Low | No timeouts or `maxDuration` on the Claude routes | New |
| 42 | frontend | Performance 3c | 🟢 Low | Main bundle weight and a PNG hero; 25 MB of assets, a third unreferenced | New |
| 43 | frontend | Code Health 4a | 🟢 Low | 33 `any` casts and a `strict: false` type-check | New |
| 44 | config | Dependencies 6 | 🟢 Low | No Dependabot, Node version undeclared, Python deps undeclared | New |
| 45 | frontend | Additional 8b | 🟢 Low | Dialogs lack focus management; low-contrast microcopy | New |
| 46 | frontend | Additional 8b | 🟢 Low | Third-party brand names in titles, OG, JSON-LD and image file names | New |
| 47 | config | Additional 8b | 🟢 Low | `SITE_URL` is the apex, which redirects every Stripe return | New |

**Total findings:** 47 (Critical: 0, High: 6, Medium: 22, Low: 19)
