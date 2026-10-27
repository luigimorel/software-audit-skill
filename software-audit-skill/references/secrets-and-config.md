# Secrets, configuration, dependencies, and crypto

Read this when auditing credentials, environment variables, deployment config, security headers, package risk, or encryption.

## Contents

- Secrets and credentials (check first, always)
- Security misconfiguration (OWASP A05)
- Vulnerable and outdated components (OWASP A06 and A08)
- Cryptography and data protection (OWASP A02)
- Deployment and environment separation
- Sources

---

## Secrets and credentials (check first, always)

The most common and most expensive vibe-coding failure.

**Look for:**
- API keys, tokens, passwords, private keys, DB connection strings in source, config, Dockerfiles, CI files, notebooks, and mobile bundles
- `.env`, `.env.local`, `*.pem`, `*.p12`, `google-services.json` with server keys, `serviceAccount*.json`, `keystore` files committed to git
- Secrets in **git history** even if deleted now (`git log -p -S "sk_live"`, `git log --all --diff-filter=D --name-only | grep -i env`)
- **Server secrets exposed to the client bundle:** in Next.js any `NEXT_PUBLIC_*`, in Vite any `VITE_*`, in Expo any `EXPO_PUBLIC_*`, and CRA `REACT_APP_*` ships to every visitor. Only publishable keys belong there.
- Supabase `service_role` key, Stripe `sk_`, OpenAI/Anthropic/Gemini keys, Firebase Admin credentials, Twilio auth tokens, AWS secret keys in client code
- Hardcoded admin passwords, "temporary" backdoor routes, default credentials
- `.gitignore` missing `.env*`, or `.env` files already tracked (`git ls-files | grep -i env`)
- `.env.example` / `.env.sample` files containing real values instead of placeholders
- Connection strings with embedded credentials (`postgresql://user:password@host`)
- Client-prefixed variables whose names suggest a secret (`NEXT_PUBLIC_..._SECRET`, `VITE_..._SERVICE_KEY`)

**Starter searches** (adapt to the stack):

```bash
# likely secrets
grep -rInE "(api[_-]?key|secret|token|passwd|password|private[_-]?key)\s*[:=]\s*['\"][^'\"]{8,}" \
  --exclude-dir={node_modules,.git,dist,build,.next,vendor} .

# well-known key formats
grep -rInE "(sk_live_|sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{35}|ghp_[A-Za-z0-9]{36}|xox[baprs]-|eyJ[A-Za-z0-9_-]{20,}\.eyJ)" \
  --exclude-dir={node_modules,.git} .

# env vars that ship to the browser
grep -rInE "NEXT_PUBLIC_|VITE_|EXPO_PUBLIC_|REACT_APP_" --exclude-dir={node_modules,.git} .
```

**Fix pattern:** rotate the key first (assume it is compromised), move it to server-side env vars or a secrets manager, proxy third-party calls through your backend, scrub history if the repo is or was public (`git filter-repo` or BFG), and add secret scanning (`gitleaks`, GitHub secret scanning) to CI.

## Security misconfiguration (OWASP A05)

**Look for:**
- **CORS:** `Access-Control-Allow-Origin: *` with credentials, or reflecting any origin
- Missing or weak headers: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors` / `X-Frame-Options`
- Debug mode in production (`DEBUG=True`, `NODE_ENV` not `production`, verbose frameworks)
- Default credentials, open admin panels, exposed databases (Mongo/Redis/Postgres on `0.0.0.0` with no auth), open S3/GCS buckets
- CSRF protection missing on cookie-authenticated state-changing routes
- HTTP instead of HTTPS, no HTTP-to-HTTPS redirect, mixed content
- Overly broad IAM roles, wildcard permissions, long-lived cloud keys
- Containers running as root; secrets baked into Docker images
- Cloud storage uploads that allow any file type, any size, served from your main domain

## Vulnerable and outdated components (OWASP A06 and A08)

**Run what is available:** `npm audit` / `pnpm audit`, `pip-audit`, `govulncheck ./...`, `cargo audit`, `osv-scanner`, `flutter pub outdated`, `./gradlew dependencyCheckAnalyze`.

**Look for:**
- Known-vulnerable or abandoned packages; unpinned `latest` versions; no lockfile
- **Hallucinated or typosquatted packages:** AI assistants sometimes suggest package names that do not exist or are look-alikes. Verify unfamiliar dependencies exist, are maintained, and have real download history before trusting them.
- Postinstall scripts from unknown packages; scripts loaded from CDNs without Subresource Integrity
- CI that deploys on any pull request with secrets access (supply chain)
- Unsigned or unverified updates (mobile OTA update channels, auto-updaters)

## Cryptography and data protection (OWASP A02)

- Sensitive data unencrypted at rest (backups, DB columns for IDs or health info, uploaded documents)
- Homemade encryption, ECB mode, hardcoded keys or IVs, `Math.random()` / `math/rand` for tokens, IDs, or OTPs (use `crypto.randomUUID`, `crypto/rand`, `secrets`)
- TLS verification disabled (`rejectUnauthorized: false`, `InsecureSkipVerify: true`, `NSAllowsArbitraryLoads`, `usesCleartextTraffic="true"`)
- PII collected "just in case" with no retention or deletion path; no way for a user to delete their account and data
- Production data copied into dev/staging or used in AI prompts without need

## Deployment and environment separation

**Look for:**
- **`.git` reachable in production.** If `https://<site>/.git/HEAD` returns content, the full source and every secret ever committed are downloadable.
- **Source maps and debug pages in production** (they expose source, stack traces, env values, and internal paths).
- **Preview deployments using production keys.** Preview URLs are often reachable by anyone who has the link. Use separate Production / Preview / Development environment variables (Vercel or equivalent); previews and local dev get test or staging keys and databases only.
- **No baseline headers.** A reasonable starting set, to loosen only where the app needs it (CDNs, inline styles):

  ```
  Content-Security-Policy: default-src 'self'; script-src 'self'
  Strict-Transport-Security: max-age=31536000; includeSubDomains
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=()
  ```

- **CORS** that allows `*` on authenticated endpoints, or pairs `Access-Control-Allow-Credentials: true` with a wildcard or reflected origin. Whitelist your own origins only.

**Pre-ship run:** `gitleaks detect` for history, confirm `.env*` is ignored, confirm debug and verbose logging are off, trigger an error page and confirm no stack trace, and confirm CORS is locked to your domains.

## Sources


- [gitleaks](https://github.com/gitleaks/gitleaks)
- [OWASP Top 10 A05: Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [GitHub Security Advisories](https://github.com/advisories)
- [OSV vulnerability database](https://osv.dev/)
- [NIST National Vulnerability Database](https://nvd.nist.gov/)
- [Next.js security advisories](https://github.com/vercel/next.js/security/advisories)
- [Composer audit documentation](https://getcomposer.org/doc/03-cli.md#audit)
