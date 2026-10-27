# Rate limiting, abuse prevention, and billing protection

Read this for any app with login, OTP, AI/LLM calls, email or SMS sending, file processing, or any endpoint that costs money per call. AI assistants almost never add rate limits on their own.

## Contents

- Where limits are required
- How to store and key the counters
- Billing protection
- Implementation pattern
- Sources

---

## Where limits are required

- **Auth endpoints:** login, signup, password reset, OTP verification, magic links, "resend code". Without limits, attackers brute-force passwords and OTPs or enumerate accounts.
- **AI/LLM calls:** one user can burn a month of budget in minutes.
- **Email and SMS sending:** an unlimited endpoint turns your app into a spam relay and runs up provider bills.
- **File processing:** upload, resize, convert, OCR. CPU-heavy work without limits is an easy denial of service.
- **Webhook-like or public ingest endpoints** that accept external input at scale.

## How to store and key the counters

- **Never keep counters in a table the client can write to.** If they live in a public Supabase table, users can reset their own limit through the REST API. Use Redis (for example Upstash), a table in a non-exposed schema, gateway or edge middleware, or in-memory counters for a single server only (Redis once you have more than one instance).
- **Combine per-IP and per-user limits.** IP-only limits fall to rotating IPs; user-only limits fall to throwaway accounts. Use both.
- Return `429` with a `Retry-After` header and a clear message.
- Count attempts per target (per account or per phone number), not only per source IP, for OTP and reset flows.

## Billing protection

- Set billing alerts on every cloud and hosting provider (AWS, GCP, Vercel, and so on).
- Set **hard spending caps** at AI providers; also enforce per-user quotas in your own database. Soft warnings alone do not stop a drain.
- Watch for anomalies: sudden spikes, odd-hour traffic, one account generating most of the cost.

## Implementation pattern

```ts
// Example: sliding-window limit with Upstash Redis
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'), // 10 requests per minute
});

export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success } = await ratelimit.limit(ip);
  if (!success) return new Response('Too many requests', { status: 429 });
  // ... handle request
}
```

Note: `x-forwarded-for` is only trustworthy behind a proxy you control; take the client IP from the platform's trusted header, not blindly from the request.


## Rate-limiter implementation and dependency CVEs

Audit both the application design and the actual limiter package/version. A correctly configured limiter can still be bypassable because of IP parsing, proxy trust, key generation, distributed-store behavior, or framework-specific routing.

**Inspect**
- Resolve exact direct and transitive versions from lockfiles. For Node.js check `express-rate-limit`, `rate-limiter-flexible`, `@upstash/ratelimit`, framework middleware/adapters, and any custom limiter. For PHP check Composer packages providing throttling, API middleware, authentication, and cache/Redis integration.
- Verify that the package's current upstream advisories and ecosystem audit results have been checked. Example: [CVE-2026-30827 / GHSA-46wh-pxpv-q5gq](https://github.com/express-rate-limit/express-rate-limit/security/advisories) describes IPv4-mapped IPv6 addresses bypassing per-client limiting in affected `express-rate-limit` configurations. Confirm the advisory's affected range and remediation against the installed version and official advisory.
- In Next.js, check the framework's own current advisories for middleware/proxy bypasses and denial-of-service issues. Middleware-based throttling or authorization is not sufficient if alternate route forms or framework vulnerabilities can bypass it; enforce limits and authorization at the API/data boundary too.
- Trust forwarded client-IP headers only when set/sanitized by a known proxy. Misconfigured proxy trust can let clients spoof their key; incorrect IPv6 normalization can group clients together or let them evade limits.
- Use shared atomic counters for multi-instance deployments. Test race conditions, fail-open/fail-closed behavior during Redis outages, key expiry, and whether users can reset or poison counters.
- Use multiple dimensions: IP/network, authenticated account, sensitive target (email/phone/account), and operation/cost. Avoid making the limiter key depend solely on an unverified user-supplied identifier.
- Apply quotas before expensive work begins, with separate limits for concurrency, payload size, timeouts, and daily/monthly spend. Return `429` and `Retry-After` where suitable.

**Run ecosystem checks**
```bash
npm audit
pnpm audit
yarn npm audit
composer audit
```
Run only the commands supported by the repository's package manager. Review the lockfile and advisory output rather than assuming a clean result covers runtime configuration or custom code.

## Sources


- [Upstash Ratelimit](https://github.com/upstash/ratelimit-js)
- [express-rate-limit security advisories](https://github.com/express-rate-limit/express-rate-limit/security/advisories)\n- [Next.js security advisories](https://github.com/vercel/next.js/security/advisories)\n- [OWASP API Security Top 10: Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/)
