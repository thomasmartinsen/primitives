---
applyTo: '**/*'
description: 'Security and authentication rules'
---

# Security guidelines

Assume the system is production-facing and potentially exposed to the internet.

---

## Secrets and sensitive data

- NEVER hardcode API keys, tokens, passwords, connection strings, or private keys.
- NEVER commit secrets to the repository — including `.env` files, `appsettings.*`, Dockerfiles, Terraform/Bicep, GitHub Actions workflows, and shell scripts.
- Use secure alternatives: environment variables, Azure Key Vault, managed identity, or OIDC.
- Treat any discovered secret as real — recommend rotation and secure storage immediately.
- Watch for long base64 or hex strings that resemble credentials.

---

## Authentication and authorization

- Microsoft Entra ID authentication is required.
- Backend API endpoints MUST validate JWT Bearer tokens issued by Entra ID.
- MCP server endpoints MUST require Bearer tokens.
- Every endpoint MUST have authorization checks — do not rely on client-side enforcement alone.
- Validate role and ownership on the server; watch for IDOR patterns (direct object reference without ownership checks).
- Cookies MUST set `Secure`, `HttpOnly`, and `SameSite` attributes.
- Tokens MUST be validated for issuer, audience, and expiry.
- Least privilege MUST be enforced for all service accounts and managed identities.

---

## Injection and unsafe input handling

- Validate all inputs at system boundaries: HTTP requests, file uploads, queues/messaging, CLI arguments, and environment variables.
- NEVER build SQL or NoSQL queries via string concatenation — use parameterised queries.
- NEVER execute shell commands using user-supplied input.
- Guard against path traversal in all file handling operations.
- Guard against SSRF — validate and restrict outbound URLs.
- Guard against template injection and deserialization of untrusted input.
- Encode all output to prevent XSS — never render unsanitised user content.

---

## Cryptography usage

- Do NOT use weak algorithms: MD5, SHA1, DES, RC4, or ECB mode.
- Do NOT implement custom cryptographic solutions — use modern, vetted libraries.
- Do NOT hardcode salts or encryption keys.
- Use cryptographically secure random number generation (`RandomNumberGenerator` in .NET, `crypto.getRandomValues()` in JS).
- Do NOT bypass TLS certificate validation.

---

## Dependencies and supply chain

- Pin dependency versions in `package.json`, `.csproj`, and other manifest files.
- Keep dependencies up to date — flag very outdated packages.
- Audit for suspicious install scripts in third-party packages.
- GitHub Actions MUST pin action versions to a specific SHA or tag.
- Do NOT download and execute remote scripts in build steps without verification.

---

## Misconfiguration

- Do NOT enable debug or development settings in production configurations.
- Do NOT expose verbose error output or stack traces to clients — use the global exception handling middleware.
- CORS policies MUST be restrictive — allow only known origins.
- Docker containers MUST run as non-root users.
- Cloud IAM permissions in IaC MUST follow least-privilege — no broad wildcard grants.
- Do NOT expose unnecessary network ports or services.

---

## Architecture enforcement

- Domain entities MUST NOT be exposed through API responses — use DTOs.

---

## Security review expectations

When introducing or modifying security-sensitive code, every finding should include:

- **Location** — file path and line.
- **Description** — what the issue is.
- **Risk** — why it matters and a plausible exploit scenario.
- **Fix** — a concrete code or configuration change.

Classify issues as: **Critical**, **High**, **Medium**, or **Low**.
