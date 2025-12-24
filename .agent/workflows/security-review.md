---
description: Deep code review focused on security vulnerabilities (OWASP-aligned)
---

# Security Code Review Workflow

This workflow guides a comprehensive security-focused code review aligned with OWASP best practices.

## Prerequisites

- Access to the full codebase
- Understanding of the application's architecture (Express server, Next.js frontend, monorepo structure)

---

## 1. Environment & Secrets Audit

// turbo
```bash
# Search for hardcoded secrets, API keys, passwords
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" -E "(password|secret|apikey|api_key|token|credential)" packages/ --exclude-dir=node_modules
```

**Manual checks:**
- [ ] Review `packages/core/src/utils/env.ts` for proper validation of all secrets
- [ ] Verify `SECRET_KEY` is 64-char hex and never logged
- [ ] Confirm no secrets in `.env.sample` have real values
- [ ] Check that `LOG_SENSITIVE_INFO` defaults to `false` in production

---

## 2. Input Validation & Injection Prevention

// turbo
```bash
# Find potential SQL injection points (raw queries)
grep -rn --include="*.ts" -E "(query\(|execute\(|raw\()" packages/ --exclude-dir=node_modules

# Find regex patterns that may be vulnerable to ReDoS
grep -rn --include="*.ts" "new RegExp" packages/ --exclude-dir=node_modules
```

**Manual checks:**
- [ ] All user inputs from `req.params`, `req.body`, `req.query` are validated
- [ ] URL parsing uses `new URL()` with try-catch
- [ ] Regex patterns in `env.ts` use `envalid` validators
- [ ] Check `packages/server/src/routes/` for unvalidated inputs

---

## 3. Authentication & Session Security

**Files to review:**
- `packages/server/src/middlewares/userData.ts` - User authentication
- `packages/server/src/middlewares/alias.ts` - Alias handling
- `packages/core/src/utils/env.ts` - `ADDON_PASSWORD`, `AIOSTREAMS_AUTH`

**Checklist:**
- [ ] UUID validation uses proper regex `/^[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}$/i`
- [ ] Password decryption errors don't leak timing information
- [ ] `TRUSTED_UUIDS` are validated before granting regex filter access
- [ ] Session tokens (if any) have proper expiry

---

## 4. Authorization & Access Control

// turbo
```bash
# Find authorization checks
grep -rn --include="*.ts" -E "(isAdmin|isTrusted|TRUSTED_UUIDS|ADDON_PASSWORD)" packages/ --exclude-dir=node_modules
```

**Checklist:**
- [ ] `REGEX_FILTER_ACCESS` properly restricts regex usage (none/trusted/all)
- [ ] Admin endpoints require authentication
- [ ] IDOR vulnerabilities: verify users can only access their own configs
- [ ] Built-in proxy auth (`AIOSTREAMS_AUTH`) validates credentials

---

## 5. Rate Limiting & DoS Protection

**Files to review:**
- `packages/server/src/middlewares/ratelimit.ts`

**Checklist:**
- [ ] All API endpoints have rate limiters applied
- [ ] Rate limit configs are sensible (check `DISABLE_RATE_LIMITS` is false in prod)
- [ ] `RECURSION_THRESHOLD_LIMIT` prevents infinite loops (default: 60 req/10s window)
- [ ] Redis store used for multi-instance deployments

---

## 6. Data Protection & Cryptography

// turbo
```bash
# Search for encryption/decryption usage
grep -rn --include="*.ts" -E "(encrypt|decrypt|crypto|randomBytes)" packages/ --exclude-dir=node_modules
```

**Checklist:**
- [ ] `SECRET_KEY` uses `crypto.randomBytes(32).toString('hex')`
- [ ] `INTERNAL_SECRET` is randomly generated on startup
- [ ] `ENCRYPT_MEDIAFLOW_URLS` and `ENCRYPT_STREMTHRU_URLS` default to `true`
- [ ] No sensitive data in URL query parameters

---

## 7. Error Handling & Logging

**Files to review:**
- `packages/server/src/middlewares/errors.ts`
- `packages/core/src/utils/logger.ts` (if exists)

**Checklist:**
- [ ] Stack traces not exposed to clients in production
- [ ] Error messages don't leak internal paths or configs
- [ ] `LOG_SENSITIVE_INFO=false` hides API keys from logs
- [ ] All security events (failed auth, rate limits) are logged

---

## 8. Dependency Security

// turbo
```bash
# Audit npm dependencies for vulnerabilities
pnpm audit --audit-level=moderate
```

**Additional checks:**
- [ ] Run `pnpm outdated` to identify outdated packages
- [ ] Review `pnpm-lock.yaml` for suspicious packages
- [ ] Check that critical security patches are applied

---

## 9. CORS & HTTP Security Headers

**Files to review:**
- `packages/server/src/middlewares/cors.ts`
- `packages/server/src/app.ts`

**Checklist:**
- [ ] CORS origin is properly restricted (not `*` in production)
- [ ] Security headers set (Content-Security-Policy, X-Frame-Options, etc.)
- [ ] HTTPS enforced via `BASE_URL` validation

---

## 10. API Security

// turbo
```bash
# List all route handlers
find packages/server/src/routes -name "*.ts" -exec basename {} \;
```

**For each endpoint, verify:**
- [ ] Authentication required where appropriate
- [ ] Rate limiting applied
- [ ] Input validation performed
- [ ] Response doesn't leak sensitive data
- [ ] HTTP methods are correct (GET for reads, POST for mutations)

---

## 11. Third-Party Integration Security

**Items to check:**
- [ ] Debrid API keys stored encrypted in database
- [ ] External URLs validated before fetch (`urlOrUrlList` validator)
- [ ] Proxy credentials (`FORCE_PROXY_CREDENTIALS`) handled securely
- [ ] OAuth flows (Google Drive) use proper state validation

---

## 12. Reporting

After completing the review:

1. Document all findings with severity (Critical/High/Medium/Low)
2. Create issues for each vulnerability
3. Prioritize fixes based on:
   - Exploitability
   - Impact
   - Affected users

**Template:**
```markdown
## [SEVERITY] Vulnerability Title

**Location:** `path/to/file.ts:lineNumber`
**Type:** (e.g., Injection, Auth Bypass, Information Disclosure)
**Description:** What the issue is
**Impact:** What could happen if exploited
**Remediation:** How to fix it
```
