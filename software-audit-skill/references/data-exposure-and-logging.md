# Data exposure and logging

Read this when auditing API responses, error output, debug endpoints, logs, analytics, crash reporting, and session replay.

## Contents

- Excessive data exposure
- Sensitive data in logs and telemetry (OWASP A09 plus privacy)
- Sources

---

## Excessive data exposure

APIs that return "the whole row" because it was easiest.

**Look for:**
- Responses that include `password`, `passwordHash`, `salt`, `resetToken`, `apiKey`, `stripeCustomerId`, internal flags, other users' emails/phones, full address, or admin notes
- `SELECT *` / ORM `findMany()` returned directly to the client; serializing whole model objects
- List endpoints with no pagination or limits (one request dumps the table)
- User search/profile endpoints that leak email, phone, or exact location to anyone
- GraphQL introspection and verbose schemas on in production; no depth/complexity limits; resolvers without field-level auth
- Verbose error messages and stack traces returned to the client (SQL errors, file paths, framework versions)
- Debug endpoints (`/debug`, `/_next/data`, `/swagger`, `/graphql` playground, `/actuator`, `/.git`, `/phpinfo`) reachable in production
- Source maps published in production builds that reveal source and comments
- Sensitive data in URLs/query strings (tokens, emails, IDs), which land in logs, history, and referrers
- Public storage buckets or directory listings
- Returning more data than the screen needs "in case the frontend wants it"
- Full DB rows passed as props to React/Next.js Client Components (the serialized props ship to the browser); select columns explicitly and use `import 'server-only'` in data-access modules

**Fix pattern:** return explicit DTOs / response types. Select only needed columns. Paginate. Add a test asserting sensitive fields never appear in responses.

```ts
// Bad
return Response.json(user);

// Good
return Response.json({ id: user.id, name: user.name, avatarUrl: user.avatarUrl });
```

```go
// Go: dedicated response struct, never the DB model
type PublicUser struct {
    ID     string `json:"id"`
    Name   string `json:"name"`
    Avatar string `json:"avatar_url"`
}
```

## Sensitive data in logs and telemetry (OWASP A09 plus privacy)

**Never log:** passwords, tokens (access, refresh, session, reset, magic-link), API keys, full card numbers or CVV, national IDs, full request/response bodies on auth or payment routes, OTP codes, private keys, health data, precise location, full emails/phones when not necessary.

**Look for:**
- `console.log(req.body)`, `console.log(user)`, `log.Printf("%+v", req)`, `print(response)`, `debugPrint`, `Log.d(...)`, `NSLog`
- Logging full headers (includes `Authorization` and `Cookie`)
- HTTP client/ORM debug logging left on in production (`logging: true`, `DEBUG=*`, `axios` interceptors that dump everything)
- Error handlers that log the full request or the full user object
- Analytics/crash tools (Sentry, Crashlytics, Mixpanel, PostHog, LogRocket, Hotjar) capturing form inputs, URLs with tokens, or session replays without masking
- Logs shipped to third parties with no retention limit
- **The opposite problem (A09):** no logs at all for logins, failures, permission denials, password resets, payment events, and admin actions; no alerting; logs that attackers can inject into (newlines in user input)
- Logs containing enough data to hijack accounts if the logging service is breached

**Fix pattern:** structured logging with a redaction list, log IDs not content, mask when needed, keep retention short, and keep a separate **audit log** for security-relevant events (who, what, when, from where, outcome) with no secrets in it.

```ts
const REDACT = ["password", "token", "authorization", "cookie", "secret", "otp", "cardNumber"];
// pino example
const logger = pino({ redact: { paths: REDACT.flatMap(k => [k, `*.${k}`, `req.headers.${k}`]), censor: "[redacted]" } });

logger.info({ userId: user.id, event: "login_success" }); // IDs, not content
```

```go
// Go (slog): log what happened, not what was sent
slog.Info("login_failed", "user_id", userID, "ip", ip) // not: "password", pw
```

## Sources

- [OWASP API Security Top 10: Broken Object Property Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa3-broken-object-property-level-authorization/)
- [OWASP Top 10 A09: Security Logging and Monitoring Failures](https://owasp.org/Top10/A09_2021-Security_Logging_and_Monitoring_Failures/)
