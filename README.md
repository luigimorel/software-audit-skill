# Software Audit Skill - Security Audits for AI Coding Assistants

An agent skill that helps AI coding assistants find common security and privacy issues before they ship, including exposed secrets, broken access control, insecure data handling, and risky dependencies.

Built by [Luigi Morel](https://github.com/luigimorel), software engineer and independent consultant at [Krafted](https://krafted.africa). 

## What it does

This standalone skill audits web and mobile applications for security and privacy issues: exposed secrets, broken access control, JWT/session flaws, dependency CVEs, unsafe database policies, sensitive logging, injection, missing rate limits, business-logic abuse, payment weaknesses, and insecure mobile storage.

Designed for solo developers and small teams. Findings are prioritized by real-world impact, supported by file/line evidence, explained in plain language, and paired with practical fixes.

1. Scopes the app, stack, data, integrations, and entry points.
2. Loads relevant reference guides based on the detected stack and risk areas.
3. Checks application logic and dependency versions, including transitive packages and current CVE/advisory data.
4. Produces a prioritized report with evidence, fixes, verification steps, positive observations, and unverified items.

The default workflow is read-only. It does not run exploits, brute-force authentication, or touch production systems. It does not change files unless explicitly asked. It never reproduces full secrets.

## Use

Install the skill in your project's `.agents/skills/audit-skill/` directory. Keep `SKILL.md` and the `references/` directory together so the skill can load its guides.

### Manual installation

From your project root, clone this repository into the skill directory:

```sh
git clone https://github.com/luigimorel/software-audit-skill.git .agents/skills/audit-skill
```

The installed layout should look like this:

```text
.agents/
└── skills/
    └── audit-skill/
        ├── SKILL.md
        └── references/
```

Restart or refresh your coding agent if it does not detect the skill immediately.

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
├── CONTRIBUTING.md
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
