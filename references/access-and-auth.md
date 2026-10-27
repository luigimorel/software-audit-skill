# Access control, authentication, and sessions

Read this when auditing who can see or change what: API routes, admin features, login, JWTs, cookies, password reset, OAuth.

## Contents

- Broken access control (OWASP A01)
- Authentication and sessions (OWASP A07)
- Next.js and server-rendered traps (middleware, Server Actions, client components)
- Sources

---

## Broken access control (OWASP A01)

AI-generated code loves to check "is the user logged in" and forget "is this *their* data".

**Look for:**
- **IDOR:** endpoints like `GET /api/orders/:id` or `/api/users/:id` that fetch by ID without checking ownership or tenant
- Authorization enforced only in the UI (hidden buttons, client-side route guards) but not on the API
- Admin routes or actions protected by a client-side flag, a cookie the user can edit, or `role` taken from the request body or JWT payload without verification
- **Mass assignment:** spreading `req.body` straight into a DB update/create, letting users set `role`, `isAdmin`, `price`, `userId`, `emailVerified`
- Missing auth middleware on newly added routes (compare route list against protected list)
- Multi-tenant apps missing a tenant filter on every query
- Server actions / API routes / Go handlers / Cloud Functions callable without auth
- Predictable IDs combined with no ownership check
- Signed URLs or share links that never expire
- Auth enforced in exactly one layer (usually edge middleware) and nowhere else (see the Next.js section below)

**Fix pattern:** enforce authorization on the server, in one central place (middleware, policy layer, or DB row-level security). Scope every query by the authenticated user or tenant. Use allow-lists for writable fields.

```ts
// Bad: trusts the caller
const order = await db.order.findUnique({ where: { id: params.id } });

// Good: ownership enforced in the query
const order = await db.order.findFirst({
  where: { id: params.id, userId: session.user.id },
});
if (!order) return new Response("Not found", { status: 404 });
```

## Authentication and sessions (OWASP A07)

**Look for:**
- Hand-rolled auth or crypto when a vetted provider/library was available
- Passwords hashed with MD5/SHA-1/plain SHA-256 or stored in plaintext (want Argon2id, bcrypt, or scrypt)
- JWTs: `alg: none` accepted, weak or hardcoded secret, no expiry, no audience/issuer check, stored in `localStorage` (XSS-stealable) rather than httpOnly cookies, no revocation path
- **`jwt.decode()` used where `jwt.verify()` is needed.** `decode` only reads the payload and never checks the signature, so anyone can forge claims such as `role: "admin"`. Pin the allowed algorithms (`algorithms: ['HS256']`) and check issuer, audience, and expiry, not just the signature.
- Session cookies missing `HttpOnly`, `Secure`, `SameSite`; session IDs not rotated on login; no logout invalidation
- No rate limiting or lockout on login, signup, OTP, password reset, and "resend code" endpoints
- Password reset tokens that are guessable, never expire, are reusable, or leak via the response
- OTPs of 4 digits with unlimited attempts; magic links that do not expire
- User enumeration ("no account with that email" vs "wrong password")
- Missing MFA on admin accounts
- OAuth: missing `state`/PKCE, loose redirect URI matching, trusting an unverified email claim
- Account takeover via email change without re-authentication


## JWT-specific verification checklist

Audit the complete token lifecycle, not just the signing call.

**Creation and validation**
- Identify every JWT library and version from the lockfile, including wrappers and JWKS/key-fetch packages.
- Verify signatures on protected routes. Decoding is not verification; reject unsigned tokens and pin an explicit algorithm allow-list appropriate to the key type. Never allow algorithm selection to be controlled by an untrusted token header.
- Validate `iss`, `aud`, `exp`, and `nbf` as applicable. Check claim types and required claims; apply a small, deliberate clock tolerance.
- Prevent algorithm/key confusion (for example, mixing symmetric and asymmetric keys), weak secrets, fallback keys, and accepting tokens from unintended issuers or tenants.
- For JWKS, restrict trusted issuer and key-source URLs; prevent attacker-controlled `jku`/`x5u` fetches, SSRF, poisoned caches, and stale-key acceptance. Validate issuer before trusting a fetched key and bound cache lifetimes.
- Do not put secrets or unnecessary personal data in JWT payloads. A signed JWT is usually readable, not encrypted.
- Keep access tokens short-lived. Define refresh-token rotation, reuse detection, logout/revocation, password-change invalidation, and key-rotation behavior. If immediate revocation is required, use a server-side session/version/denylist strategy rather than assuming a stateless JWT can be revoked.
- Ensure authorization uses verified claims and current server-side entitlements where stale roles, disabled accounts, or revoked access matter.
- Check token storage and transport: prefer secure, `HttpOnly`, `Secure`, appropriately `SameSite` cookies for browser sessions when compatible with the architecture; protect cookie-authenticated mutations against CSRF. Avoid logging tokens or placing them in URLs.

**Node.js package and CVE checks**
- Inspect `jsonwebtoken`, `jose`, `jws`, `fast-jwt`, `jwks-rsa`, `get-jwks`, and any authentication framework's JWT adapters when present. These are examples, not an exhaustive list.
- Run the package manager audit and inspect transitive dependency paths. Confirm advisory affected ranges against the resolved lockfile version and upstream security advisory.
- Examples of advisories to check when the corresponding package is used include [CVE-2025-65945 (`jws`)](https://nvd.nist.gov/vuln/detail/CVE-2025-65945), [CVE-2025-59936 (`get-jwks`)](https://nvd.nist.gov/vuln/detail/CVE-2025-59936), and [CVE-2025-30144 (`fast-jwt`)](https://nvd.nist.gov/vuln/detail/CVE-2025-30144). Verify current affected/fixed ranges from the primary advisory; do not infer vulnerability from package name alone.
- Treat library-specific CVE checks as separate from application-level JWT misconfiguration. A patched package can still be used unsafely, and safe application code cannot repair a vulnerable package implementation.

## Next.js and server-rendered traps (middleware, Server Actions, client components)

AI-generated Next.js code tends to assume the UI is the only caller. It is not.

**Look for:**
- **Middleware as the only wall.** Edge middleware is a convenience layer, not a sole auth boundary. CVE-2025-29927 showed it could be skipped entirely with a spoofed `x-middleware-subrequest` header. Auth must be re-checked inside Server Actions, route handlers (`app/api/`), and the data-access layer.
- **Unprotected Server Actions.** A `'use server'` function compiles to a public POST endpoint that anyone can call with `curl`. Each one needs, at the top: runtime input validation (Zod or similar), authentication, and an ownership check on the resource.
- **Whole DB rows passed to Client Components.** Passing a full user object into a client component ships hashes, internal flags, and admin fields to the browser. Select only the needed columns. Mark data-access modules with `import 'server-only'` so they cannot be imported client-side by accident.
- **Tokens in `localStorage`.** Prefer `HttpOnly; Secure; SameSite=Lax` cookies, which scripts cannot read, so a single XSS no longer exposes every session.

```ts
// Bad: public endpoint with no validation, auth, or ownership check
'use server';
export async function deleteItem(id: string) {
  await db.items.delete({ where: { id } });
}

// Good: validate, authenticate, authorize (ownership in the query)
'use server';
export async function deleteItem(input: unknown) {
  const parsed = schema.safeParse(input);
  if (!parsed.success) return { error: 'Invalid input' };

  const session = await auth();
  if (!session?.user) redirect('/login');

  await db.items.deleteMany({
    where: { id: parsed.data.id, userId: session.user.id },
  });
}
```

```ts
// JWT: verify, do not just decode
const payload = jwt.verify(token, secret, { algorithms: ['HS256'], issuer: 'your-app' });
```

## Sources

- [CVE-2025-29927 (Next.js middleware authorization bypass, NVD)](https://nvd.nist.gov/vuln/detail/CVE-2025-29927)
- [Next.js security advisories](https://github.com/vercel/next.js/security/advisories)
- [Node.js jsonwebtoken security advisories](https://github.com/auth0/node-jsonwebtoken/security/advisories)
- [GitHub Advisory Database](https://github.com/advisories)
- [OWASP Top 10 A01: Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [OWASP Top 10 A07: Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
