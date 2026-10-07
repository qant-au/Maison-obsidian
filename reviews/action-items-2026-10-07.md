# Maison Obsidian Action Items - 2026-10-07

Derived from [code-review-2026-10-07.md](./code-review-2026-10-07.md). Review framework by Adam Burgess - https://github.com/qant-au. Items are listed in execution order: Critical first, then High, then Medium grouped by theme, then Low. Each item is independently actionable and references its finding in the review by appendix number (e.g. `#4` -> row 4 of the Appendix Finding Summary Table) and originating section.

**This is the single working action-items document.** Every prior `action-items-*.md` (all of them in `reviews/history/`) is now fully closed - items still open in them were re-verified at HEAD and carried into this file. Do not re-open the older files; add new work here.

Priority key: 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low

---

## 🔴 Critical (do first)

None this cycle.

---

## 🟠 High (do before next release)

### 1. Require a confirmed email before matching guest orders to an account
- **Why:** `commits_select_own` hands any signed-in user every unclaimed guest order whose `user_email` equals the email claim in their JWT, and the setup guide suggests turning email confirmation off. Anyone who learns a guest's address can sign up as it and read their name, street address, phone and purchases.
- **What:** Rewrite the email branch of the policy to join `auth.users` and require `email_confirmed_at is not null`, mirroring `can_review()` in `0033`; or replace the branch with a `SECURITY DEFINER` `my_orders()` RPC that performs the check. Update `store.ts:fetchMyCommits` to match. Add to `docs/SUPABASE_SETUP.md` that *Confirm email* must be on in production.
- **Where:** `supabase/migrations/0018_guest_orders.sql:11-21`, `src/lib/store.ts:133-149`, `docs/SUPABASE_SETUP.md:119-121`
- **Refs:** Review #1 (Section 2a)

### 2. Authorise or delete the `create-shipment` Edge Function
- **Why:** Any holder of the public anon key can create `shipments` rows for any commit and, if the Australia Post credentials are set, buy real labels. Nothing in the app calls it.
- **What:** Delete the function and its setup-guide instructions, or add a server-side `is_admin()` check via the bearer token (as `isAdminRequest()` does), restrict `shipTo` to the commit's stored address, pin the `esm.sh` import to an exact version. Run `supabase functions list` to learn whether it is currently deployed and remove it if so.
- **Where:** `supabase/functions/create-shipment/index.ts:25,133-186`, `docs/SUPABASE_SETUP.md:233-276`
- **Refs:** Review #2 (Section 2a)

### 3. Rotate the Supabase secret key sitting in `SUPABASE_ANON_KEY` and replace it with the publishable key
- **Why:** The production serverless environment feeds a secret key to every "anon" client in `api/`; the site's own `/api/stripe/status` has been reporting `checkoutReady: false` for this reason, publicly.
- **What:** In Vercel, set `SUPABASE_ANON_KEY` to the `sb_publishable_` key and redeploy; confirm `/api/stripe/status` reports `checkoutReady: true`. Rotate the secret key in the Supabase dashboard and update `SUPABASE_SERVICE_ROLE_KEY` wherever the rotated value is used.
- **Where:** Vercel production environment; `api/_lib/stripe.ts:31-33`
- **Refs:** Review #3 (Section 2d)

### 4. Add security headers to every response
- **Why:** The live site serves no CSP, no frame-ancestors, no nosniff and no Referrer-Policy while the Supabase session sits in `localStorage`; one injected script takes every customer's session, and the site can be framed by any origin.
- **What:** Add a `headers` block to `vercel.json` for `/(.*)`: Content-Security-Policy (start with the policy drafted in the review and tighten from the console), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`, and HSTS with `includeSubDomains; preload`. Verify with `curl -sI` after deploy.
- **Where:** `vercel.json`
- **Refs:** Review #4 (Section 2e)

### 5. Rate-limit and cache the checkout and shipping-quote routes
- **Why:** Both are unauthenticated and uncounted; each call spends an Australia Post PAC request and, for checkout, Stripe calls. Exhausting the PAC quota blocks postal checkout for every real customer.
- **What:** Shared-store rate limiter (Upstash Redis or a Supabase table keyed by IP+minute) on `/api/shipping/quote` and `/api/stripe/checkout`; cache PAC results per (bag shape, postcode) for an hour; add a WAF rule on `/api/` if the plan allows; alert on PAC quota.
- **Where:** `api/shipping/quote.ts:15-47`, `api/stripe/checkout.ts:17-179`, `api/_lib/auspost.ts:73-115`
- **Refs:** Review #5 (Section 2e)

### 6. Publish the privacy policy, returns policy and support contact
- **Why:** The live site collects names, addresses, phones, photographs and chat transcripts and takes live payments with no privacy notice (APP 1/5) and no returns terms, while promising "30-day returns" in the bag, checkout and concierge.
- **What:** Write and host a privacy policy (processors: Supabase, Stripe, Resend, Google Analytics, Anthropic; photographs; transcripts; retention; access/deletion) and a returns/refunds policy consistent with ACL guarantees; set `VITE_SUPPORT_EMAIL`, `VITE_PRIVACY_POLICY_URL`, `VITE_RETURNS_POLICY_URL` in Vercel and redeploy; link the privacy policy from checkout, the footer sign-up, the Scent DNA lead form and the Scent Memory upload. Until published, remove the "30-day returns" copy.
- **Where:** `src/components/Help.tsx:4-21`, `src/components/BagDrawer.tsx:153,159`, `src/components/Checkout.tsx:491-492`, `api/chat.ts:26`, `src/lib/concierge.ts:136`
- **Refs:** Review #6 (Section 2i)

---

## 🟡 Medium - Security hardening

### 7. Enforce VIP-only on the server or remove the feature
- **Why:** `buyable()` ignores `vipOnly`, so checkout sells VIP bottles to anyone; `enroll_subscriber` and a leftover `with check (true)` policy let any visitor make themselves VIP for free; the concierge quotes a $120/yr price nothing collects.
- **What:** Decide. If VIP stays: check `vip_only` against `subscribers` in `priceLines()`/`subscribe.ts`, drop `subscribers_insert`, gate enrolment behind a paid Stripe product. If not: remove `vip_only`, the VIP copy in `api/chat.ts` and `concierge.ts`, and the `/about` enrolment; record the decision in the instructions file's Settled positions.
- **Where:** `api/_lib/catalogue.ts:112-117`, `api/stripe/subscribe.ts:33-40`, `supabase/migrations/0001_init.sql:167-188,211-213`, `src/App.tsx:111-114,352-359`
- **Refs:** Review #7 (Section 2a)

### 8. Stop the client cancelling a Stripe-billed subscription in the database only
- **Why:** When the cancel route is unreachable the client falls through to the `cancel_subscription` RPC; the row reads "Cancelled" while Stripe keeps charging. The RPC is also callable directly.
- **What:** Revoke `cancel_subscription` from `authenticated`; make `cancelSubscription()` fail loudly when the route returns null instead of falling back.
- **Where:** `supabase/migrations/0012_subscriptions.sql:118-133`, `src/lib/subscription.ts:270-286`
- **Refs:** Review #8 (Section 2a)

### 9. Build the AI system prompts on the server
- **Why:** `chat` and `scent-ai` take the catalogue (and chat takes the taste profile) from the request body and put it in the system prompt, making both an open Claude proxy with attacker-controlled instructions in the house's name.
- **What:** Move `catalogueContext()`/`catalogueSummary()` to `api/_lib/` and build them from `loadCatalogue()`; ignore any client `catalogue`; derive `profile` from the bearer token server-side. Keep the cache breakpoint.
- **Where:** `api/chat.ts:36-47,136`, `api/scent-ai.ts:403,415`, `src/lib/scentai.ts:112-133`, `src/lib/concierge.ts:62-72`
- **Refs:** Review #9 (Section 2c)

### 10. Replace the per-instance rate limiter on the AI routes with a shared one and add spend ceilings
- **Why:** The in-memory counter resets with every instance; `scent-ai` sends up to 7 MB images and 8,192 Opus tokens per anonymous call.
- **What:** Shared-store limiter per IP per minute plus a daily cap and a global daily ceiling; require a session for the `imagine` image operation; set `maxDuration` and an Anthropic client `timeout`; configure an Anthropic spend alert.
- **Where:** `api/chat.ts:50-78`, `api/scent-ai.ts:53-77,304-311`, `api/conceive.ts:135-159`
- **Refs:** Review #10 (Section 2c)

### 11. Untrack `.env` and move the browser configuration into Vercel
- **Why:** The file is tracked despite the ignore rule, so the next real key pasted into it is committed; production's browser Supabase config is currently coming from this file rather than from Vercel (`VITE_SUPABASE_ANON_KEY: MISSING` in the live status while the bundle has the key).
- **What:** `git rm --cached .env`; set `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` in Vercel for all environments; redeploy and confirm the bundle still carries the publishable key.
- **Where:** `.env`, Vercel project settings
- **Refs:** Review #11 (Section 2d)

### 12. Gate `/api/stripe/status` behind admin, or strip it for anonymous callers
- **Why:** It currently tells anyone that Stripe is live, which key kinds sit in which slot (including the misconfiguration), and the postcode the business posts from.
- **What:** Return 404 unless `isAdminRequest()` passes; or keep only `checkoutReady` for unauthenticated callers.
- **Where:** `api/stripe/status.ts:39-103`
- **Refs:** Review #12 (Section 2e)

### 13. Stop inlining the whole `import.meta.env` object and remove the test publishable key
- **Why:** `socialProfiles(import.meta.env)` makes Vite ship every `VITE_*` variable, which today includes a `pk_test_` Stripe key (unused) and Vercel metadata, and would ship any future `VITE_`-prefixed secret automatically.
- **What:** Read the four social URL keys explicitly in `social.ts`; pass a plain object from `prerender.mjs`; delete `VITE_STRIPE_PUBLISHABLE_KEY` from Vercel.
- **Where:** `src/lib/social.ts:21`, `scripts/prerender.mjs:198`, `src/App.tsx:389`, `src/components/Footer.tsx:27`
- **Refs:** Review #15 (Section 2f)

### 14. Double opt-in for marketing and waitlist sign-ups
- **Why:** Any typed address is recorded as having consented with no confirmation; Spam Act consent must come from the address holder, and strangers can sign third parties up.
- **What:** Send a confirmation via Resend with a signed token and set `opted_in`/`confirmed_at` only when followed; add a working unsubscribe link to the waitlist and recovery emails.
- **Where:** `supabase/migrations/0014_profiles_marketing.sql:58-79`, `0035_waitlist.sql:42-69`, `src/components/Footer.tsx:49-54`, `api/_lib/waitlist.ts:53-70`, `api/_lib/recovery.ts:58-78`
- **Refs:** Review #17 (Section 2i)

---

## 🟡 Medium - Payments

### 15. Rewrite the payment documentation to match the immediate-charge model and delete the authorise-later stub
- **Why:** The README describes card holds, batch capture and a `capture-batch` function that do not exist; the TODO file contradicts the code on `allow_promotion_codes` and the pinned API version.
- **What:** Rewrite README payment/batch sections; delete `authorizePayment()` and `VITE_STRIPE_AUTHORIZE_URL`; correct the two TODO rows (promotion codes `true`; API version `2026-08-26.dahlia`).
- **Where:** `README.md:6-9,317-323,491-510`, `src/lib/stripe.ts:1-57`, `STRIPE_INTEGRATION_TODO.md:31,39`, `.env.example`
- **Refs:** Review #13 (Section 2f)

### 16. Remove the admin "Bill month N" button and its revoked-RPC call chain
- **Why:** It calls a stub payment id and a revoked RPC, discards the error and reloads, so it looks like it worked.
- **What:** Remove the button, `authorizePayment`, `billSubscriptionMonth` and `startSubscription` from the client.
- **Where:** `src/components/AdminSubscriptions.tsx:50-59,132-136`, `src/lib/subscription.ts:201-242,305-324`, `src/lib/stripe.ts:37-57`
- **Refs:** Review #14 (Section 2f)

---

## 🟡 Medium - Architecture & code health

### 17. Make seed and price migrations production-safe and replace the demo inventory in production
- **Why:** `0003`/`0006` restock any zero row with demo figures, `0027`/`0028` reset every price, `0002`/`0021` overwrite admin edits; the setup guide says they are safe to re-run. Production still carries the demo stock/oil (`f14`: 24/12/6, 600 ml) and shows it as "Ready to ship".
- **What:** Move demo stock/oil into a separate never-in-production seed; mark price migrations one-shot; correct `docs/SUPABASE_SETUP.md:75-79`; have the admin set real stock and oil (or zero) on every fragrance.
- **Where:** `supabase/migrations/0003_admin_inventory.sql:35-37`, `0006_oil_inventory.sql:12`, `0027_house_prices.sql`, `0028_house_price_30ml.sql`, `0021_audience_catalogue.sql:58-82`, `docs/SUPABASE_SETUP.md:75-79`
- **Refs:** Review #16 (Section 2h)

### 18. Delete the legacy payment paths and surface admin RPC errors
- **Why:** `recordCommit` and `startSubscription` are uncalled and target revoked RPCs; every `admin.ts` helper returns `!error` and callers discard it, so a refused admin action looks like success.
- **What:** Delete the dead functions; return error messages from the admin helpers and show them in the console (the `saveError` pattern exists).
- **Where:** `src/lib/store.ts:89-112`, `src/lib/subscription.ts:201-242`, `src/lib/admin.ts:95-130,141-169,240-259`
- **Refs:** Review #18 (Section 4a)

### 19. Add a parity test between `src/lib/formats.ts` and `api/_lib/catalogue.ts`
- **Why:** Pricing rules are duplicated per runtime; the comment claiming the route test compares them is false. A change in one silently diverges displayed price from charged price.
- **What:** Extend `test_checkout_security.mjs` to transpile both modules and assert `formatPrice`, `subscriptionPrice` and `buyable` agree for every seed fragrance and format; fix the comment.
- **Where:** `api/_lib/catalogue.ts:1-5`, `scripts/test_checkout_security.mjs`
- **Refs:** Review #19 (Section 4a)

### 20. Fail visibly when the live catalogue cannot be read in production
- **Why:** A failed read renders the 56-entry seed (retired scents, demo stock, stale prices) with no indication, while the live catalogue has 59 rows with launch dates and real images. The README documents the fallback as deliberate; that was right for a demo and is not right for a live store.
- **What:** In configured mode, retry then show a "catalogue temporarily unavailable" state; keep the seed for `!supabase` only.
- **Where:** `src/lib/store.ts:57-80`, `README.md:641-643`
- **Refs:** Review #20 (Section 4a)

---

## 🟡 Medium - Testing

### 21. Cover the webhook signature path, subscription lifecycle and migrations with tests, and run `test:security` automatically
- **Why:** The one assertion suite is not run by anything; webhook rejection, renewals, the Resend senders, RLS and the migrations are untested. The function-count gate is at its ceiling.
- **What:** Add `test:security` to `build` or CI; add a forged-signature webhook test; add a `supabase start` migration test with `set role anon` assertions; note the plan for the 13th route.
- **Where:** `scripts/test_checkout_security.mjs`, `package.json:8`, `api/stripe/webhook.ts:36-43`
- **Refs:** Review #21 (Section 4c)

---

## 🟡 Medium - Configuration & operational

### 22. Reconcile `.env.example` and the setup guide with the variables the code reads, and make the mail senders fail closed
- **Why:** Resend variables are undocumented; Mailgun variables are documented but unread; an operator following the docs configures the wrong provider and the reminder/launch emails are silently skipped.
- **What:** Regenerate `.env.example` and the reference table from the env-read grep; document `RESEND_API_KEY`, `RECOVERY_EMAIL_FROM`, `RECOVERY_REPLY_TO`, `SUPABASE_URL`, `SUPABASE_ANON_KEY`; remove `MAILGUN_*` and `VITE_STRIPE_AUTHORIZE_URL`; make `sendRecoveryEmail`/`waitlistNotify` report "not configured" to the admin UI.
- **Where:** `.env.example`, `docs/SUPABASE_SETUP.md:137-211,408-428`, `api/_lib/recovery.ts:47-52`, `api/_lib/waitlist.ts:26-31`
- **Refs:** Review #22 (Section 4d)

### 23. Add error tracking, a webhook-failure alert and a real health check
- **Why:** Webhook 500s go only to function logs; nobody would notice `record_paid_order` failing until a customer asked.
- **What:** Sentry or a log drain with an alert on `api/stripe/*` 5xx; Stripe's failed-webhook email; store `event.id` on `processed_checkout_sessions`; a `/api/health` that checks the DB and Stripe, watched by an uptime monitor.
- **Where:** `api/stripe/webhook.ts:85-88`, `api/_lib/record.ts:119`, `supabase/migrations/0029_checkout_security.sql:40-46`
- **Refs:** Review #23 (Section 4e)

### 24. Track migrations, document backups and write the four runbooks
- **Why:** Migrations are pasted by hand with no record of what ran; no rollback, backup or restore note; no runbooks for webhook, Australia Post, Anthropic or Resend failures.
- **What:** Link the project and apply via `supabase db push` from CI; document the backup tier and rehearse one restore; add `docs/runbooks/` for the four failures; note the 12-function ceiling.
- **Where:** `docs/SUPABASE_SETUP.md:59-85`, `supabase/migrations/0028_house_price_30ml.sql:8-11`
- **Refs:** Review #25 (Section 7)

### 25. Make the build type-check strict and cover `api/`, and lint `api/`
- **Why:** `tsc -b` runs the non-strict `tsconfig.json` only; the strict app config is dead and fails; `api/` is neither type-checked nor linted by any automatic step.
- **What:** Turn `tsconfig.json` into a solution file with `references` to app, api and node configs; fix the three strict errors in `staffDesk.ts`; remove `api` from the ESLint ignore and add a Node-globals block.
- **Where:** `tsconfig.json`, `tsconfig.app.json`, `tsconfig.api.json`, `eslint.config.js:9`, `src/lib/staffDesk.ts:119,126,139`
- **Refs:** Review #26 (Section 7b)

### 26. Add CI, a pre-push hook and a secret scan
- **Why:** Every gate except the two inside `npm run build` runs only when a person remembers; there is no secret scan on a repository with a tracked `.env`.
- **What:** One GitHub Actions workflow: `npm ci`, `lint`, `tsc -p tsconfig.api.json`, `test:security`, `build`, `npm audit --omit=dev --audit-level=high`, gitleaks; mirror as a `pre-push` hook; require the check in Vercel.
- **Where:** `.github/workflows/ci.yml` (new), `package.json`
- **Refs:** Review #27 (Section 7b)

---

## 🟡 Medium - Documentation & dependency hygiene

### 27. Rewrite the stale README sections and the setup guide's provider and function instructions
- **Why:** Eight named mismatches between README/docs and the tree (batch model, staff passphrase, missing components, migration list, Mailgun, `capture-batch`, "batch commits" in the chat greeting, TODO rows).
- **What:** Rewrite payment, staff desk, architecture and migration sections from the code; replace Mailgun with Resend; delete `capture-batch` instructions; fix `ChatWidget.tsx` greeting and suggestion copy; fix the two TODO rows; add the settled positions to `reviews/code-review-instructions.md`.
- **Where:** `README.md`, `docs/SUPABASE_SETUP.md`, `STRIPE_INTEGRATION_TODO.md`, `src/components/ChatWidget.tsx:18-21`
- **Refs:** Review #24 (Section 5)

### 28. Add `shippingDetails` and `hasMerchantReturnPolicy` to Product offers
- **Why:** Search Console flags every Offer without them; the workspace gates these fields.
- **What:** Extend `offerFor()` with `OfferShippingDetails` (AU, free over $100, else rate range) and `MerchantReturnPolicy` once item 6's returns policy exists; derive sitemap `lastmod` from a real date or drop it.
- **Where:** `src/lib/seo.ts:158-167`, `scripts/prerender.mjs:212-218`
- **Refs:** Review #28 (Section 8b)

---

## 🟢 Low (do when convenient)

### 29. Revoke the default `PUBLIC` execute grant on every pre-0025 function
- **What:** One migration that `revoke execute … from public` on every function and re-grants to `authenticated`/`anon` as intended; confirm with `supabase db lint`.
- **Where:** `supabase/migrations/0001` through `0024`
- **Refs:** Review #29 (Section 2a)

### 30. Normalise and strip control characters from engraving and free-text order fields; guard CSV cells
- **What:** NFC-normalise and strip `\p{C}` in `priceLines()` and `checkout.ts`; prefix CSV cells beginning with `=`, `+`, `-`, `@`.
- **Where:** `api/_lib/stripe.ts:223`, `api/stripe/checkout.ts:52-56`, `src/lib/staffDesk.ts:222-228`
- **Refs:** Review #30 (Section 2b)

### 31. Return upstream Claude error text only to admins
- **What:** Log `apiDetail()` server-side; include it in the response only when `isAdminRequest()` passes.
- **Where:** `api/scent-ai.ts:442-443`, `api/conceive.ts:335-336`, `api/marketing.ts:129-130`
- **Refs:** Review #31 (Section 2c)

### 32. Set `SITE_URL` to the `www` origin and fail hard without it in production
- **What:** Vercel `SITE_URL=https://www.maisonobsidian.com.au`; in `siteUrl()`, throw when `VERCEL_ENV === "production"` and `SITE_URL` is unset, keeping the header fallback for previews.
- **Where:** `api/_lib/stripe.ts:76-82`, Vercel production env
- **Refs:** Review #33, #47 (Sections 2e, 8b)

### 33. Treat a null confirmation as an error state on the thank-you page
- **What:** In configured mode show `status: "error"` with the "if your card was charged the order is safe" copy and keep the bag until a confirm succeeds.
- **Where:** `src/App.tsx:298-302`
- **Refs:** Review #34 (Section 2f)

### 34. Apply `npm audit fix` for the dev-tree advisories
- **What:** `npm audit fix`, commit the lockfile, rebuild and run the gates.
- **Where:** `package-lock.json`
- **Refs:** Review #35 (Section 2g)

### 35. Column-grant `fragrances` so `oil_ml` and `low_stock_threshold` are not public, and select named columns
- **What:** `revoke select` then `grant select (cols)` to `anon`/`authenticated` excluding the two internal columns; name columns in `store.ts` and `prerender.mjs`; give admin an RPC for the hidden ones.
- **Where:** `supabase/migrations/0001_init.sql:196-198`, `src/lib/store.ts:61`, `scripts/prerender.mjs:120`
- **Refs:** Review #36, #39 (Sections 2h, 3a)

### 36. Bound the payloads of the anonymous write RPCs
- **What:** Validate `dims`/`adjustments` keys against the sixteen dimensions and clamp values 0-100 inside `save_scentprint` and `record_scent_signal`; cap `jsonb` size.
- **Where:** `supabase/migrations/0020_scent_dna_wearer.sql:18-73`, `0024_scent_signals.sql:45-88`
- **Refs:** Review #37 (Section 2h)

### 37. Add retention and a deletion runbook for customer data
- **What:** `pg_cron` deletes for anonymous `chat_messages` (30 days) and ownerless profiles/signals (12 months); a documented account-deletion procedure covering `commits` PII.
- **Where:** `supabase/migrations/` (new), `docs/` (new)
- **Refs:** Review #38 (Section 2i)

### 38. Index `commits.payment_intent_id`
- **What:** `create index if not exists commits_payment_intent_idx on public.commits(payment_intent_id);`
- **Where:** `supabase/migrations/` (new); used by `api/_lib/record.ts:223,231`
- **Refs:** Review #40 (Section 3a)

### 39. Set timeouts and `maxDuration` on the Claude routes
- **What:** `new Anthropic({ apiKey, timeout: 45_000, maxRetries: 1 })` and `maxDuration: 60` on `chat`, `scent-ai`, `conceive`.
- **Where:** `api/chat.ts:15,126`, `api/scent-ai.ts:23,405`, `api/conceive.ts:17,292`
- **Refs:** Review #41 (Section 3b)

### 40. Trim the asset folder, convert the hero to WebP and add intrinsic image sizes
- **What:** Delete or relocate the ~36 unreferenced spares (including the 6.8 MB `oud wood.png`); convert `bottle-pair.png` and the per-slug PNGs to WebP; add `width`/`height` to `BottleImage`.
- **Where:** `public/assets/`, `src/components/BottleImage.tsx:48,63,75`, `scripts/trim_bottle_renders.py`
- **Refs:** Review #42 (Section 3c)

### 41. Reduce `any` usage in `api/` once the strict type-check lands
- **What:** Type `req`/`res` with `@vercel/node`'s `VercelRequest`/`VercelResponse`; remove the remaining casts.
- **Where:** `api/**/*.ts`
- **Refs:** Review #43 (Section 4a)

### 42. Declare the Node version, add Dependabot and a Python requirements file
- **What:** `engines.node` + `.nvmrc` matching Vercel; `.github/dependabot.yml` (npm weekly, grouped minor/patch); `scripts/requirements.txt` with `openpyxl` and `pillow`.
- **Where:** `package.json`, `.nvmrc` (new), `.github/dependabot.yml` (new), `scripts/requirements.txt` (new)
- **Refs:** Review #44 (Section 6)

### 43. Add focus management to the four dialogs and raise microcopy contrast
- **What:** A shared `useDialog` hook (initial focus, Tab trap, Escape, restore focus) for `AuthModal`, `PasswordReset`, `BagDrawer`, `ChatWidget`; `aria-labelledby` on headings; microcopy alpha ≥ 0.6; run axe on checkout and Scent DNA.
- **Where:** `src/components/AuthModal.tsx:150-187`, `src/components/PasswordReset.tsx:59-80`, `src/components/BagDrawer.tsx:69`, `src/components/ChatWidget.tsx:108-128`, `src/components/styles.ts` (`micro`)
- **Refs:** Review #45 (Section 8b)

### 44. Get a legal read on third-party brand names in titles and the provenance of brand-named photographs
- **What:** Review the `productTitle()`/OG/JSON-LD pattern against comparative-advertising norms; confirm the 24 brand-named files in `public/assets/` are the house's own photography; surface the independence disclaimer on the rendered product page, not only the crawler snapshot.
- **Where:** `src/lib/seo.ts:149-155`, `src/lib/snapshot.ts:37`, `public/assets/`
- **Refs:** Review #46 (Section 8b)
- _Confidence: Low - a legal question, not a code defect._

---

*End of action items. See [code-review-2026-10-07.md](./code-review-2026-10-07.md) for full context, severity definitions, and the executive summary.*
