---
name: audit-skill
description: Security and privacy audit for web and mobile apps built fast with AI assistance ("vibe coded"). Checks for OWASP Top 10 issues, leaked secrets, broken access control, sensitive data in logs, APIs that expose too much data, misconfigured Supabase/Firebase rules, insecure mobile storage, and risky dependencies. Use this skill whenever the user asks to audit, review, harden, pen-test, sanity-check, or "make safe" an app or repo, asks "is my app secure?", "is this safe to launch/ship?", "am I leaking anything?", or mentions OWASP, MASVS, RLS, exposed API keys, or logging sensitive data, rate limiting, Stripe webhooks, or Next.js Server Actions, even if they don't say the word "audit". Also use before a launch, after a big AI-generated feature, or when reviewing a React/Next.js/Vue/Node/Go/Flutter/React Native/Swift/Kotlin codebase for security.
---

# Vibe App Audit

Audit web and mobile apps for the security and privacy mistakes that fast, AI-assisted development produces most often. The reader is probably a solo builder or small team who shipped quickly and cannot afford a 40-page pentest report. Be accurate, prioritized, and practical.

## Core rule: never trust the client

Every price, user ID, role, subscription status, feature flag, and rate-limit counter must be validated or enforced on the server. Anything that exists only in the browser, the mobile bundle, or the request body is attacker-controlled. Most findings below are this one rule broken in a different place.

## Principles

1. **Find what actually gets people hurt first.** Leaked secrets, missing authorization, and over-exposed data matter far more than a missing header. Order findings by real-world impact, not by checklist order.
2. **Plain language.** Explain each issue as "what could happen to your users or your bill", then show the fix. Define jargon (IDOR, RLS, CORS) in one short phrase the first time it appears.
3. **Every finding gets a fix.** Include a copy-pasteable code or config change in the user's stack, not generic advice.
4. **Evidence over vibes.** Cite file paths and line numbers. Never report an issue you did not see in code, config, or a response. If you could not verify something, label it "Unverified" and say what to check.
5. **Read-only by default.** Do not run exploits, brute force, fuzz, or hit production systems. Only audit code and systems the user owns or is authorized to test. Never modify files unless asked.
6. **Never reproduce secrets.** When you find a key or token, show only the first 4 and last 2 characters (e.g. `sk_l…9f`) and tell the user to rotate it. Do not paste full credentials into the report.
7. **Acknowledge what is done well.** A short "Good" section helps the user know what not to break.

## Workflow

### Step 1: Scope and recon (2 minutes)

Identify before judging:

- **Platforms:** web, iOS, Android, cross-platform (React Native, Flutter, Expo, Capacitor)
- **Stack:** frontend framework, backend (Node, Go, Python, serverless), database, auth provider, BaaS (Supabase, Firebase, Appwrite, PocketBase), hosting (Vercel, Cloudflare, Netlify, AWS)
- **Third parties:** payments (Stripe, Flutterwave, Paystack, mobile money), email/SMS, analytics, AI APIs, storage buckets
- **Data handled:** PII, payment data, health data, children's data, location, photos, messages
- **Entry points:** run `ls`, read `package.json` / `go.mod` / `pubspec.yaml` / `Podfile` / `build.gradle`, routing files, `.env.example`, deployment config

If the user gave only a description or pasted code, audit what you have and list what you would need to see for full coverage (e.g. "database policies", "mobile `AndroidManifest.xml`").

### Step 2: Run the checks

Work through the categories in the table below in priority order, reading the matching reference file for each. Use grep and file reads liberally. Skip categories that clearly do not apply (e.g. mobile checks for a pure web app) and say so.

### Step 3: Write the report

Use the report template at the bottom. Lead with a verdict and the top issues.

---

## Check categories and where to find them

Read the reference file for each category that applies. Do not load all of them by default: pick based on the recon in Step 1. Start with secrets, always.

| #   | Category                                                                                 | Reference file                                                          | Read when                                                                                |
| --- | ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 1   | Secrets and credentials                                                                  | [secrets-and-config.md](references/secrets-and-config.md)               | Always, first                                                                            |
| 2   | Broken access control (IDOR, mass assignment, Server Actions, middleware)                | [access-and-auth.md](references/access-and-auth.md)                     | Any app with users or an API                                                             |
| 3   | Backend-as-a-service rules (Supabase, Firebase, Convex)                                  | [baas-rules.md](references/baas-rules.md)                               | Client talks directly to a DB or storage                                                 |
| 4   | Excessive data exposure                                                                  | [data-exposure-and-logging.md](references/data-exposure-and-logging.md) | Any API, GraphQL, or server-rendered UI                                                  |
| 5   | Sensitive data in logs and telemetry                                                     | [data-exposure-and-logging.md](references/data-exposure-and-logging.md) | Any app with logging, analytics, or crash reporting                                      |
| 6   | Injection (SQL, ORM operator, XSS, command, SSRF-adjacent) and input validation          | [injection-and-abuse.md](references/injection-and-abuse.md)             | Any user input reaches a query, a shell, HTML, or a URL                                  |
| 7   | Authentication and sessions (JWT, cookies, OAuth, reset)                                 | [access-and-auth.md](references/access-and-auth.md)                     | Any login, JWT, cookie, OAuth, or reset flow                                             |
| 8   | Security misconfiguration and deployment (CORS, headers, debug, env separation)          | [secrets-and-config.md](references/secrets-and-config.md)               | Any deployed app                                                                         |
| 9   | Vulnerable and outdated components, including CVEs in direct and transitive dependencies | [secrets-and-config.md](references/secrets-and-config.md)               | Any app with dependencies                                                                |
| 10  | Cryptography and data protection                                                         | [secrets-and-config.md](references/secrets-and-config.md)               | App stores or transmits sensitive data                                                   |
| 11  | Business logic, abuse, and SSRF                                                          | [injection-and-abuse.md](references/injection-and-abuse.md)             | Uploads, webhooks, user-supplied URLs, email/SMS                                         |
| 12  | Rate limiting and billing protection                                                     | [rate-limiting.md](references/rate-limiting.md)                         | Login/OTP, AI calls, email/SMS, file processing, paid APIs                               |
| 13  | Money, pricing, and payments (incl. Stripe)                                              | [money-and-pricing.md](references/money-and-pricing.md)                 | Anything that charges, refunds, credits, or keeps balances                               |
| 14  | LLM integrations                                                                         | [integrations.md](references/integrations.md)                           | App sends data to or receives output from an LLM provider                                |
| 15  | Third-party API integrations                                                             | [integrations.md](references/integrations.md)                           | App calls an external API, embeds a provider SDK, or accepts provider callbacks/webhooks |
| M   | Mobile (iOS, Android, RN, Flutter, Expo)                                                 | [mobile.md](references/mobile.md)                                       | Any mobile build, in addition to the rows above                                          |

Skip categories that clearly do not apply and say so in the report's "Not reviewed" line.

### Fast path

If the user only has a few minutes, skip the references and run the pre-launch gate at the bottom of this file, then offer a full audit.

---

## Severity rubric

| Level        | Meaning                                            | Examples                                                                                                                                       |
| ------------ | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Critical** | Exploitable now, serious harm, little skill needed | Live secret key in public repo/client bundle; RLS off on a table with user data; unauthenticated admin endpoint; SQL injection; payment bypass |
| **High**     | Likely exploitable, significant harm               | IDOR on user records; passwords hashed with MD5; JWT in localStorage plus an XSS; tokens in logs; no rate limit on OTP                         |
| **Medium**   | Needs conditions or chaining                       | Permissive CORS; missing CSP; verbose errors; outdated dependency with a non-trivial CVE; unprotected deep link parameter                      |
| **Low**      | Hardening or hygiene                               | Missing `Referrer-Policy`; source maps public; no certificate pinning in a low-risk app                                                        |
| **Info**     | Observation or good practice                       | Suggest enabling secret scanning, dependency bot, MFA                                                                                          |

When unsure between two levels, pick the higher one and explain what would lower it.

---

## Report template

Produce the report in Markdown using this structure. Keep it skimmable.

````markdown
# Security & Privacy Audit: <app name>

**Scope:** <what was reviewed: repo/paths, platforms, commit or date>
**Stack:** <short summary>
**Not reviewed:** <anything out of reach: production config, infra, third-party dashboards>

## Verdict

<One short paragraph: ship / ship after fixing X / do not ship. Count of findings by severity.>

| Critical | High | Medium | Low | Info |
| :------: | :--: | :----: | :-: | :--: |
|    0     |  0   |   0    |  0  |  0   |

## Fix these first

1. **<Title>** (Critical): one-line impact and where it is.
2. ...
3. ...

## Findings

### [C-1] <Short title> (Critical)

- **Where:** `path/to/file.ts:42`
- **What's wrong:** <plain-language explanation>
- **What could happen:** <concrete impact for users or the business>
- **Fix:**
  ```ts
  // minimal corrected code or config
  ```
- **Verify the fix:** <how to confirm, e.g. a curl command, a test, a dashboard check>
- **Reference:** <link from the matching reference file's Sources section, e.g. OWASP page or provider docs>

### [H-1] ...

## What's already good

- <e.g. parameterized queries throughout, RLS enabled on all tables>

## Hardening checklist (nice to have)

- [ ] Secret scanning in CI
- [ ] Dependency update bot
- [ ] Security headers
- [ ] Audit log for logins and admin actions
- [ ] Account and data deletion flow

## Unverified items

- <things that need access you did not have, and exactly what to check>
````

---

## Style rules for the report

- Use IDs (`C-1`, `H-2`, `M-3`) so the user can refer to findings in follow-up prompts.
- One finding per issue; group repeated instances ("same pattern in 7 routes: list them") instead of 7 separate findings.
- Keep fixes minimal and idiomatic to the user's stack and version. Do not suggest a rewrite when a 3-line change works.
- Rank by impact, not by how easy the finding is to explain.
- Do not pad with theoretical risks. If the app is in good shape, say so briefly.
- Offer to apply the fixes afterward ("Want me to patch the Critical and High items?") but do not change files unprompted.
- If you find a live secret, put **rotate it now** at the top of the report before anything else.

## Quick pre-launch gate

If the user only has a few minutes, run just these eight checks:

1. Any secret in code, git history, or client bundle? Rotate it.
2. Can user A read or edit user B's data by changing an ID?
3. Is RLS / Firestore rules / storage policy on and scoped to the owner?
4. Does any API response include fields the screen does not need (hashes, emails, tokens)?
5. Are passwords, tokens, or full request bodies being logged?
6. Are login, OTP, reset, and AI/email/SMS endpoints rate-limited?
7. Is prod running with debug off, HTTPS only, and no test or admin routes exposed?
8. Does every Server Action, route handler, and edge function check auth and ownership itself, not only middleware or the UI?

---

## References and credits

Reference files in this skill (all in `references/`):

- [secrets-and-config.md](references/secrets-and-config.md): secrets, misconfiguration, deployment, dependencies, crypto
- [access-and-auth.md](references/access-and-auth.md): access control, authentication, sessions, Next.js traps
- [baas-rules.md](references/baas-rules.md): Supabase, Firebase, Convex
- [data-exposure-and-logging.md](references/data-exposure-and-logging.md): over-exposed responses, logs, telemetry
- [injection-and-abuse.md](references/injection-and-abuse.md): injection, validation, abuse, SSRF, LLM features
- [rate-limiting.md](references/rate-limiting.md): limits, counters, billing caps
- [money-and-pricing.md](references/money-and-pricing.md): prices, coupons, webhooks, refunds, ledgers, Stripe
- [integrations.md](references/integrations.md): third-party APIs, webhooks, provider credentials, LLM integrations
- [mobile.md](references/mobile.md): iOS, Android, React Native, Flutter, Expo
- [rule-register.md](references/rule-register.md): format for documenting and extracting business rules

Each reference ends with a **Sources** section linking the standards, docs, and upstream material it draws on. When you write a finding, copy the relevant link into its **Reference** line.

Standards used throughout:

- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP MASVS](https://mas.owasp.org/MASVS/) and [Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)

This skill is maintained as a standalone application-security audit workflow.
