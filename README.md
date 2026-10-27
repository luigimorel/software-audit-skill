# Independent AppSec Audit

A standalone agent skill for auditing web and mobile applications for security and privacy issues: exposed secrets, broken access control, JWT/session flaws, dependency CVEs, unsafe database policies, sensitive logging, injection, missing rate limits, business-logic abuse, payment weaknesses, and insecure mobile storage.

Designed for solo developers and small teams. Findings are prioritized by real-world impact, supported by file/line evidence, explained in plain language, and paired with practical fixes.

## What it does

1. Scopes the app, stack, data, integrations, and entry points.
2. Loads relevant reference guides based on the detected stack and risk areas.
3. Checks application logic and dependency versions, including transitive packages and current CVE/advisory data.
4. Produces a prioritized report with evidence, fixes, verification steps, positive observations, and unverified items.

The default workflow is read-only. It does not run exploits, brute-force authentication, or touch production systems. It does not change files unless explicitly asked. It never reproduces full secrets.

## Use

Install this folder as `audit-skill/` in the skill directory supported by your agent, or provide the folder/ZIP to an Agent Skills-compatible tool.

Example requests:

- “Audit this repository before launch.”
- “Review my JWT authentication and rate limits.”
- “Check my Next.js app and npm dependencies for known CVEs.”
- “Review PHP/Composer packages and authentication endpoints.”
- “Audit Supabase RLS, Stripe webhooks, and authorization.”
- “Run the quick pre-launch gate.”

## Contents

```text
audit-skill/
├── SKILL.md
├── README.md
├── LICENSE
└── references/
    ├── access-and-auth.md
    ├── baas-rules.md
    ├── data-exposure-and-logging.md
    ├── injection-and-abuse.md
    ├── integrations.md
    ├── money-and-pricing.md
    ├── mobile.md
    ├── rate-limiting.md
    ├── rule-register.md
    └── secrets-and-config.md
```

## Security references

- [OWASP Top 10](https://owasp.org/Top10/)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [GitHub Security Advisories](https://github.com/advisories)
- [OSV vulnerability database](https://osv.dev/)
- [NIST National Vulnerability Database](https://nvd.nist.gov/)
- [npm audit documentation](https://docs.npmjs.com/cli/commands/npm-audit)
- [Composer audit documentation](https://getcomposer.org/doc/03-cli.md#audit)
- [Next.js security advisories](https://github.com/vercel/next.js/security/advisories)
- [Node.js jsonwebtoken advisories](https://github.com/auth0/node-jsonwebtoken/security/advisories)
- [express-rate-limit advisories](https://github.com/express-rate-limit/express-rate-limit/security/advisories)

Use current advisories during each audit; this list is a starting point, not an exhaustive or permanently current CVE catalogue.
