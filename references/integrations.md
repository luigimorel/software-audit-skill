# External API and LLM integrations

Read this when an app calls a third-party API, embeds a provider SDK, accepts provider webhooks, or sends data to an LLM. Pair it with the relevant domain guide (for example, `money-and-pricing.md` for billing rules or `injection-and-abuse.md` for input handling).

## Contents

- Integration inventory and trust boundaries
- Credentials and environment separation
- Outbound API calls
- Webhooks and callbacks
- LLM provider integrations
- Data handling and provider configuration
- Sources

---

## Integration inventory and trust boundaries

For each provider, trace the full path: user input or event -> application server -> provider -> callback or response -> application state. Record the provider, SDK and version, server/client call sites, credentials, data sent, webhook routes, and what state or action the integration can change.

**Look for:**

- Provider calls made directly from a browser or mobile app when they require a private credential
- Client-supplied user IDs, tenant IDs, prices, roles, or callback destinations used to select provider resources
- Provider responses treated as authorization decisions without checking the local user's access and the expected provider account/resource
- Integration routes that skip the normal authentication, authorization, validation, or audit logging used by other routes
- A sandbox credential or test endpoint used in production, or production credentials used by local development and tests

Treat provider responses as external input. A successful provider response does not by itself prove that the current user owns the associated local record or is allowed to perform the requested action.

## Credentials and environment separation

**Look for:**

- Private API keys, signing secrets, or refresh tokens in frontend bundles, mobile packages, source control, logs, or error reports
- Credentials with broad permissions where read-only or resource-scoped credentials would work
- No rotation/revocation process, or credentials shared across unrelated apps and environments
- Provider credentials copied into URLs or query strings, which can leak through logs and referrers
- Production secrets available to preview deployments, forks, or untrusted CI jobs

Keep private credentials server-side, load them from the deployment's secret manager, scope them to the smallest provider permissions, and separate production from test credentials. Public client identifiers are not secrets, but still validate the origin, redirect URI, and allowed scopes attached to them.

## Outbound API calls

**Look for:**

- User-controlled URLs, hosts, paths, or redirect targets passed to a server-side HTTP client; use a fixed provider host or strict allow-list and re-check redirects to prevent SSRF
- Missing connection/read timeouts, response-size limits, or bounded retry limits
- Retries of non-idempotent operations (payments, messages, provisioning) without a provider-supported idempotency key
- Retry storms or unbounded concurrency that can exhaust the app's resources or provider quota
- TLS verification disabled, credentials forwarded across redirects, or sensitive query parameters
- Provider response bodies or error messages trusted without schema, type, range, and state validation
- Fail-open behavior that grants access, records payment, or completes a workflow when the provider is unavailable or returns an ambiguous result
- Provider errors or raw request/response bodies logged with credentials or personal data

Validate the response at runtime, bind it to the expected local user and operation, and make failure behavior explicit. Apply per-user and global limits to calls that consume quota or money; see `rate-limiting.md`.

## Webhooks and callbacks

**Look for:**

- Callback accepted based only on an obscure URL or untrusted event fields
- Missing signature verification, verification over parsed/re-serialized JSON instead of the provider's required raw bytes, or a signing secret shared across environments
- No timestamp/replay window where supported, or no deduplication by provider event ID
- Duplicate or out-of-order events that apply side effects more than once or roll state backward
- Event metadata used as proof of user identity, ownership, amount, or entitlement without comparing against server-created local records
- Handler doing slow work inline, returning failure after partial side effects, or acknowledging an event before durable processing

Verify signatures before processing, enforce idempotency in durable storage, validate event type and account/resource association, and make state transitions monotonic where possible. Payment-specific event and amount checks are in `money-and-pricing.md`.

## LLM provider integrations

In addition to the controls above, inspect:

- Provider keys only on the server; model, token, request, and spend limits enforced per user/tenant as well as globally
- Data sent to the model minimized and covered by the app's privacy disclosures, retention commitments, and any required opt-out or consent
- Provider retention, training, region, and subprocessors configured to match the app's data commitments; do not assume defaults meet them
- Retrieved documents filtered by the requesting user's permissions before they enter the prompt
- Prompt injection treated as untrusted input: model instructions are not an authorization boundary, and tool calls require server-side authorization on every invocation
- Tools limited to an explicit allow-list with validated argument schemas and least privilege; require confirmation for consequential actions
- Model output treated as untrusted data: validate structured output, escape rendered text, and never execute generated SQL, shell commands, or code
- Conversation history, prompts, tool arguments, and completions not retained or logged longer than necessary
- Provider fallbacks and retries not silently sending sensitive data to a different provider or model

See `injection-and-abuse.md` for prompt injection and unsafe tool-use patterns, and `rate-limiting.md` for quota and cost controls.

## Data handling and provider configuration

For each integration, identify the personal, confidential, or regulated data in requests, responses, telemetry, and provider-side storage. Check provider retention/deletion controls, data residency and subprocessors where applicable, contractual terms, and the app's user-facing disclosures. Verify that debug mode and request tracing do not capture full payloads or authorization headers.

When provider dashboards or contracts are unavailable, report the configuration as **Unverified** and state exactly what an operator must check. Do not infer the provider's retention or security settings from SDK code alone.

## Sources

- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/)
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OpenAI API data usage policies](https://platform.openai.com/docs/guides/your-data)
- [Stripe webhooks](https://docs.stripe.com/webhooks)
