---
name: security-review
description: Comprehensive security review skill that analyzes code for security vulnerabilities and best practices including input validation, injection prevention, secret handling, unsafe operations, path traversal, authentication, and OWASP Top 10 compliance. Use when performing security-focused code reviews.
---

# Security Review

Performs comprehensive security analysis of the Black Duck Security Scan codebase, identifying vulnerabilities and ensuring security best practices are followed. Always read changed files before producing findings. Never flag theoretical risks when the code pattern already mitigates them.

## Usage

Run this skill when the user requests:
- "Review code security"
- "Check for security vulnerabilities"
- "Perform security audit"
- "Security scan"
- "SAST review"
- "Check for security issues"
- "OWASP Top 10 review"

## How to Use

When conducting a security review:

1. **Read all changed files** (or files specified by user)
2. **Run each checklist** section below against the code
3. **Report findings** as: `file:line — [SEVERITY] — vulnerability — remediation`
4. **Use severity tags**: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`

---

## Severity Reference

| Tag | Meaning | Examples |
|---|---|---|
| `CRITICAL` | Direct credential exfiltration or code execution possible | Hardcoded secrets, command injection, eval() on user input |
| `HIGH` | Credential leak to logs/files, injection vulnerability, TLS bypass | Secrets in logs, path traversal, SQL injection, rejectUnauthorized: false |
| `MEDIUM` | Conditional exposure, information disclosure | Error messages with sensitive data, missing input validation |
| `LOW` | Defense-in-depth hardening, not exploitable alone | Missing rate limiting, weak cipher suites |

---

## Checklist 1 — Secret and Credential Logging

**Context:** Credentials flow through `inputs.ts` exports (`POLARIS_ACCESS_TOKEN`, `BLACKDUCKSCA_API_TOKEN`, `COVERITY_USER`, `COVERITY_PASSPHRASE`, `GITHUB_TOKEN`). These are written into JSON input files and passed to Bridge CLI.

**Checks:**
- [ ] No credential input from `inputs.ts` passed to `core.info()`, `core.debug()`, `core.warning()`, or `core.error()`
- [ ] No secret value interpolated into template literals that feed any log call, e.g. `` `token: ${inputs.POLARIS_ACCESS_TOKEN}` ``
- [ ] No `JSON.stringify(fullInputData)` passed to any log function — contains tokens/passwords
- [ ] File paths logged, never their contents when they contain secrets
- [ ] `core.setSecret()` called for all credential values to mask them in logs
- [ ] Error objects from HTTP calls do not propagate response bodies containing auth headers to logs
- [ ] No secrets in `core.setOutput()` or `core.exportVariable()` calls
- [ ] GitHub Actions annotations (`::error::`, `::warning::`) do not contain secrets

**Remediation pattern:**
```typescript
// ❌ WRONG — logs credential
core.info(`Using token: ${POLARIS_ACCESS_TOKEN}`)
console.log(`Config: ${JSON.stringify(polarisData)}`)

// ✅ CORRECT — mask secret first, log without sensitive data
core.setSecret(POLARIS_ACCESS_TOKEN)
core.info(`Connecting to server: ${POLARIS_SERVER_URL}`)
core.info(`Generated input file at: ${inputFilePath}`)
```

**CRITICAL risk:** Credentials in logs are visible to anyone with Actions read access and persist in log storage.

---

## Checklist 2 — Credential Exposure in Temporary Files

**Context:** `tools-parameter.ts` writes full product config (including access tokens) to temp files: `polaris_input.json`, `bd_input.json`, etc. These live in `os.tmpdir()` during action execution.

**Checks:**
- [ ] Temp files written to `os.tmpdir()` — NOT to workspace or hardcoded path
- [ ] Temp files not written to any path derived from user input (path traversal risk)
- [ ] Temp directory cleanup verified — `main.ts` finally block must `rmRF` the temp dir
- [ ] Temp files not uploaded as artifacts (no `@actions/artifact` upload of `*_input.json`)
- [ ] Diagnostics artifact upload does NOT include `*_input.json` files
- [ ] No `fs.writeFileSync()` with world-readable permissions (`0o644`) for secret-containing files
- [ ] File permissions on temp files are restrictive (0o600 owner-only if possible)

**Remediation pattern:**
```typescript
// ❌ WRONG — hardcoded path
const inputFile = '/tmp/polaris_input.json'

// ❌ WRONG — workspace path (persists after run)
const inputFile = path.join(process.cwd(), 'polaris_input.json')

// ✅ CORRECT — OS temp directory with cleanup
const tempDir = fs.mkdtempSync(path.join(os.tmpdir(), 'action_temp_'))
const inputFile = path.join(tempDir, 'polaris_input.json')
try {
  // Use file
} finally {
  fs.rmSync(tempDir, { recursive: true, force: true })
}
```

**HIGH risk:** If temp files persist or are in accessible locations, other processes/users can read credentials.

---

## Checklist 3 — Command Injection

**Context:** `@actions/exec` is used to execute Bridge CLI. The command is built by concatenating `--stage` and `--input` flags in `tools-parameter.ts`.

**Checks:**
- [ ] Bridge CLI executable path resolved via `path.join()` from validated base — NOT from raw user input
- [ ] No user-supplied string injected directly into command without sanitization
- [ ] Product URLs, tokens, paths go into JSON input files — NOT into command arguments
- [ ] File paths in command arguments are properly quoted if needed
- [ ] No `eval()`, `new Function()`, `child_process.execSync()`, or `child_process.spawn()` with `shell: true`
- [ ] Bridge CLI executable path validated with `fs.existsSync()` before execution
- [ ] `BRIDGECLI_INSTALL_DIRECTORY` path traversal check — resolved path stays within expected bounds
- [ ] Working directory for exec is validated (uses `process.env.GITHUB_WORKSPACE`)

**Remediation pattern:**
```typescript
// ❌ WRONG — string concatenation (shell injection risk)
exec(`bridge-cli --stage ${userInput}`)

// ✅ CORRECT — array args (no shell parsing)
await exec.exec('bridge-cli', ['--stage', 'polaris', '--input', inputPath])
```

**CRITICAL risk:** Shell injection allows arbitrary command execution on the runner.

---

## Checklist 4 — Path Traversal

**Context:** File paths are constructed in `utility.ts`, `tools-parameter.ts`, and user-controlled inputs include `BRIDGECLI_INSTALL_DIRECTORY`, temp file paths, SARIF file paths.

**Checks:**
- [ ] User-supplied path inputs passed through `path.resolve()` or `path.normalize()` before use
- [ ] Resolved path verified to stay within expected base (workspace or temp dir)
- [ ] Check for `..` traversal escaping the allowed tree
- [ ] SARIF file paths (user-overridable) resolved relative to workspace — not treated as absolute blindly
- [ ] No `__dirname` or relative path used as base for security-critical operations
- [ ] Path validation before `fs.readFileSync()`, `fs.writeFileSync()`, `fs.existsSync()`
- [ ] No file operations on paths containing `..` without normalization

**Remediation pattern:**
```typescript
// ❌ WRONG — no validation
fs.readFile(userPath)

// ✅ CORRECT — validate within allowed base
const basePath = path.resolve(process.env.GITHUB_WORKSPACE)
const userPath = path.resolve(inputs.CUSTOM_PATH)
const normalizedPath = path.normalize(userPath)

if (!normalizedPath.startsWith(basePath + path.sep) && normalizedPath !== basePath) {
  throw new Error('Path traversal detected')
}
fs.readFileSync(normalizedPath)
```

**HIGH risk:** Path traversal can read/write arbitrary files on the runner, including secrets from other jobs.

---

## Checklist 5 — TLS / SSL Security

**Context:** `ssl-utils.ts` handles custom CA certificates. Network operations use `https` module and axios.

**Checks:**
- [ ] No `rejectUnauthorized: false` without explicit user opt-in
- [ ] `NODE_TLS_REJECT_UNAUTHORIZED=0` never set in codebase
- [ ] Custom CA certificates appended to system CAs — not replacing them
- [ ] No SSL verification disabled by default
- [ ] All HTTPS requests use Node's default secure settings unless explicitly overridden
- [ ] Proxy configuration doesn't bypass TLS verification
- [ ] Certificate validation errors are not silently caught and ignored

**Remediation pattern:**
```typescript
// ❌ WRONG — TLS bypass
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0'
const agent = new https.Agent({ rejectUnauthorized: false })

// ✅ CORRECT — secure defaults
// Use Node's default https behavior
// Only customize if user explicitly provides CA cert
```

**HIGH risk:** TLS bypass enables man-in-the-middle attacks.

---

## Checklist 6 — Token Handling and Authorization

**Context:** `GITHUB_TOKEN` is used for GitHub API operations. Other tokens passed to Bridge CLI.

**Checks:**
- [ ] `Authorization` headers only sent over HTTPS endpoints
- [ ] Tokens not logged at any level — even base64 encoded
- [ ] Tokens consumed from GitHub Secrets via `core.getInput()` — NOT hardcoded or from env directly
- [ ] No token value in error messages
- [ ] `core.setSecret()` called for all tokens before any operations
- [ ] Authorization headers not forwarded to non-GitHub endpoints
- [ ] Tokens not written to files outside temp directory
- [ ] No tokens in workflow command outputs (`::set-output::`)

**Remediation pattern:**
```typescript
// ❌ WRONG — token in logs
const token = core.getInput('github_token')
core.info(`Using token: ${token}`)

// ✅ CORRECT — mask first
const token = core.getInput('github_token')
core.setSecret(token)
core.info('GitHub token configured')
```

**HIGH risk:** Exposed tokens allow unauthorized API access and repository operations.

---

## Checklist 7 — Input Injection into JSON Payloads

**Context:** User inputs (URLs, project names, assessment types) are serialized into `*_input.json` via `JSON.stringify()`.

**Checks:**
- [ ] All user inputs placed into typed model objects before `JSON.stringify()`
- [ ] Typed assignment prevents extra key injection
- [ ] Assessment type values validated against allowed list before use
- [ ] No user input concatenated directly into JSON strings (manual `"{"key": " + userVal + "}"`)
- [ ] Severity values validated against allowed list
- [ ] Branch names, project names from environment treated as untrusted
- [ ] No prototype pollution risk — objects created with `Object.create(null)` if needed

**Remediation pattern:**
```typescript
// ❌ WRONG — string concatenation
const json = `{"token": "${userToken}"}`

// ✅ CORRECT — typed object + JSON.stringify
const data: PolarisData = {
  accesstoken: userToken,
  serverUrl: userUrl
}
const json = JSON.stringify({ data: { polaris: data } })
```

**MEDIUM risk:** JSON injection can alter structure or inject unexpected fields.

---

## Checklist 8 — GitHub Actions Environment Security

**Context:** GitHub Actions exposes environment variables and allows setting outputs/variables.

**Checks:**
- [ ] Only non-sensitive data in `core.setOutput()` — exit codes/status strings are safe
- [ ] No `core.setOutput()` with secret-derived values
- [ ] No `core.exportVariable()` with secrets
- [ ] `core.setFailed()` message doesn't include credentials — only error codes
- [ ] Environment variables containing secrets treated as untrusted
- [ ] No secrets in artifact names or paths
- [ ] Action outputs documented as non-secret in action.yml

**Remediation pattern:**
```typescript
// ❌ WRONG — secret in output
core.setOutput('polaris_token', accessToken)

// ✅ CORRECT — only non-sensitive data
core.setOutput('scan_status', exitCode === 0 ? 'success' : 'failure')
```

**HIGH risk:** Outputs are visible in workflow logs and to downstream jobs.

---

## Checklist 9 — Dependency and Supply Chain Security

**Checks:**
- [ ] Dependencies use exact versions in `package.json` — not `^` or `~` ranges for security-critical deps
- [ ] `package-lock.json` committed and up to date
- [ ] `npm audit` shows no high/critical vulnerabilities
- [ ] No dependencies with known CVEs
- [ ] `dist/` directory rebuilt after dependency changes (`npm run package`)
- [ ] No eval-capable packages for untrusted input processing
- [ ] Build artifacts don't bundle `.env` files or private keys
- [ ] Dependencies regularly updated for security patches

**Remediation pattern:**
```bash
# Check for vulnerabilities
npm audit

# Fix auto-fixable issues
npm audit fix

# Update package-lock.json
npm install

# Rebuild distribution
npm run package
```

**CRITICAL risk:** Vulnerable dependencies can be exploited if they process untrusted input.

---

## Checklist 10 — Error Message Information Disclosure

**Context:** Error messages from `validators.ts`, `bridge-cli.ts`, `tools-parameter.ts` appear in GitHub Actions logs visible to repository members.

**Checks:**
- [ ] Error messages reference parameter names — NOT actual input values
- [ ] HTTP error responses don't include response body if it could contain stack traces
- [ ] File not found errors reference parameter name — not resolved file path
- [ ] No `JSON.stringify(error)` that might include request headers with tokens
- [ ] Version file parse errors don't expose full file content
- [ ] Stack traces sanitized to remove sensitive paths if logged

**Remediation pattern:**
```typescript
// ❌ WRONG — exposes value
throw new Error(`Invalid token: ${token}`)

// ✅ CORRECT — references param name only
throw new Error('polaris_access_token is required but not provided')
```

**MEDIUM risk:** Information disclosure aids attackers in reconnaissance.

---

## OWASP Top 10 Compliance Checklist

Run through this for comprehensive coverage:

### 1. Injection (A03:2021)
- [ ] Command injection prevented (no shell execution with user input)
- [ ] Path traversal prevented (path validation)
- [ ] No SQL injection risk (N/A - no database)
- [ ] JSON injection prevented (typed objects)

### 2. Cryptographic Failures (A02:2021)
- [ ] Secrets not in logs
- [ ] Secrets not in error messages
- [ ] TLS enforced (no `rejectUnauthorized: false` by default)
- [ ] Temp files cleaned up

### 3. Injection (covered above)

### 4. Insecure Design (A04:2021)
- [ ] Principle of least privilege (minimal permissions)
- [ ] Validation on all inputs
- [ ] Secure defaults

### 5. Security Misconfiguration (A05:2021)
- [ ] No secrets in repository
- [ ] Secure defaults for all settings
- [ ] Error messages don't expose internals

### 6. Vulnerable and Outdated Components (A06:2021)
- [ ] Dependencies up to date
- [ ] No known vulnerable versions
- [ ] Regular `npm audit`

### 7. Identification and Authentication Failures (A07:2021)
- [ ] Tokens from GitHub Secrets
- [ ] No hardcoded credentials
- [ ] Tokens masked with `core.setSecret()`

### 8. Software and Data Integrity Failures (A08:2021)
- [ ] Verify Bridge CLI download integrity (checksums if available)
- [ ] No eval or unsafe deserialization
- [ ] Type checking after JSON.parse

### 9. Security Logging and Monitoring Failures (A09:2021)
- [ ] Security events logged (auth failures, validation errors)
- [ ] Logs don't contain secrets
- [ ] Error handling doesn't swallow security exceptions

### 10. Server-Side Request Forgery (A10:2021)
- [ ] URLs validated before HTTP requests
- [ ] Internal network access restricted
- [ ] Proxy configuration validated

---

## Known Safe Patterns (Do Not Flag)

These patterns are secure by design — do not flag as vulnerabilities:

- ✅ `@actions/exec` with array args — not shell injection vulnerable
- ✅ `path.join(tempDir, filename)` for generated JSON files — safe if tempDir from `os.tmpdir()`
- ✅ `JSON.stringify(typedObject)` where object from typed model — not prototype pollution
- ✅ `core.setSecret(token)` before logging — correct masking pattern
- ✅ `fs.mkdtempSync(path.join(os.tmpdir(), prefix))` — secure temp directory
- ✅ Reading `process.env.GITHUB_*` variables — these are GitHub-controlled
- ✅ `@actions/core`, `@actions/exec`, `@actions/github` usage — official GitHub packages

---

## Security Scanning Process

### Step 1: Automated Scans

```bash
# Dependency vulnerabilities
npm audit

# Check for outdated packages
npm outdated

# Type check
npm run build
```

### Step 2: Pattern-Based Search

```bash
# Find potential secrets
grep -r "password\s*=\s*['\"]" src/
grep -r "api[-_]key\s*=\s*['\"]" src/
grep -r "token\s*=\s*['\"]" src/

# Find unsafe operations
grep -r "eval(" src/
grep -r "new Function" src/

# Find command execution
grep -r "exec(" src/
grep -r "spawn(" src/

# Find file operations
grep -r "fs.readFile" src/
grep -r "fs.writeFile" src/
```

### Step 3: Manual Code Review

For each security area:
1. Run automated checks
2. Apply checklist
3. Document findings
4. Prioritize by severity

---

## Security Report Format

```markdown
# Security Review Report

**Date**: [Date]
**Reviewer**: Security Review Skill
**Scope**: [Files reviewed]

## Executive Summary

### Overall Security Posture: [SECURE / VULNERABLE / CRITICAL]

- **Critical Issues**: [count]
- **High Priority**: [count]
- **Medium Priority**: [count]
- **Low Priority**: [count]

---

## Critical Issues (Fix Immediately)

### 1. [Vulnerability Name]
- **File**: `file.ts:line`
- **Category**: [Secret Handling / Injection / etc.]
- **Severity**: CRITICAL
- **Problem**: [Description]
- **Impact**: [What attacker can do]
- **Remediation**:
  ```typescript
  // Current (vulnerable)
  [bad code]

  // Fixed
  [good code]
  ```

---

## High Priority Issues

[Same format as Critical]

---

## Medium Priority Issues

[Same format]

---

## Low Priority Issues

[Same format]

---

## OWASP Top 10 Assessment

| Category | Status | Issues |
|----------|--------|--------|
| Injection | ✅/⚠️/❌ | [count] |
| Cryptographic Failures | ✅/⚠️/❌ | [count] |
| [etc] | | |

---

## Recommendations

### Immediate (This Week)
1. Fix all critical issues
2. Rotate any exposed credentials
3. [...]

### Short-term (This Month)
1. Fix high priority issues
2. Add security tests
3. [...]

### Long-term (This Quarter)
1. Automated security scanning in CI/CD
2. Regular dependency audits
3. Security training

---

## Dependencies Audit

```bash
npm audit summary:
Critical: [count]
High: [count]
Medium: [count]
Low: [count]
```

**Action Required**:
- Update [package] to [version] (fixes CVE-XXXX-YYYY)

---

**Next Steps**:
1. Address critical findings immediately
2. Create issues for high/medium
3. Schedule re-review after fixes
```

---

## Best Practices

### Review Principles
- **Risk-based**: Prioritize by impact and exploitability
- **Comprehensive**: Cover all OWASP Top 10
- **Actionable**: Provide specific remediation with code examples
- **Verifiable**: Include file:line references

### When to Run
- Before every release
- After adding new features
- When dependencies change
- After security incidents
- Weekly as part of security hygiene

### Tools to Use
- `npm audit` for dependency scanning
- `grep` for pattern-based searching
- Manual code review for logic flaws
- GitHub security alerts for vulnerable dependencies
