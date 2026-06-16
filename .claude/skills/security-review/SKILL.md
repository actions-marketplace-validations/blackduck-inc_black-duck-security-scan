---
name: security-review
description: Comprehensive security review skill that analyzes code for security vulnerabilities and best practices including input validation, injection prevention, secret handling, unsafe operations, path traversal, authentication, and OWASP Top 10 compliance. Use when performing security-focused code reviews.
---

# Security Review

Performs comprehensive security analysis of the Black Duck Security Scan codebase, identifying vulnerabilities and ensuring security best practices are followed.

## Usage

Run this skill when the user requests:
- "Review code security"
- "Check for security vulnerabilities"
- "Perform security audit"
- "Security scan"
- "Check for security issues"
- "OWASP Top 10 review"
- "Find security vulnerabilities"

## Security Review Areas

This skill covers 10 key security areas:

1. **Input Validation** - All user inputs properly validated
2. **Injection Prevention** - Command, SQL, and path injection prevention
3. **Unsafe Operations** - Avoiding eval, innerHTML, and dangerous functions
4. **Secret Handling** - Proper management of credentials and tokens
5. **Data Sanitization** - Cleaning and validating external data
6. **Authentication & Authorization** - Secure token and credential management
7. **Network Security** - HTTPS, SSL/TLS, and proxy configuration
8. **File System Security** - Path traversal and temp file handling
9. **Dependency Security** - Third-party library vulnerabilities
10. **Logging & Monitoring** - Secure logging without exposing secrets

---

## Area 1: Input Validation

### What to Check

**All User Inputs Must Be Validated**:
- Type checking for external data
- Range and length validation
- Regex validation for patterns
- Null/undefined handling

### Good Patterns

**From validators.ts**:
```typescript
/**
 * Validates inputs and returns array of errors
 */
export function validateInputs(): string[] {
  const errors: string[] = []

  // Validate required fields
  if (!inputs.SERVER_URL) {
    errors.push('Server URL is required')
  }

  // Validate format/pattern
  if (inputs.SERVER_URL && !isValidUrl(inputs.SERVER_URL)) {
    errors.push('Server URL must be a valid HTTPS URL')
  }

  return errors
}
```

### Bad Patterns

```typescript
// ❌ DANGEROUS: No validation
function process(userInput: string) {
  return exec(`command ${userInput}`)
}

// ❌ DANGEROUS: Missing null check
function process(input: string | null) {
  return input.toUpperCase() // Error if null
}

// ❌ DANGEROUS: Direct use without validation
const command = process.env.USER_COMMAND
exec(command)
```

### Search For

```bash
# Find unvalidated inputs
grep -rn "core.getInput" src/ | grep -v "validate"
grep -rn "process.env" src/ | grep -v "GITHUB_"
```

### Report Items

```markdown
### Input Validation Issues

**Issue**: Unvalidated user input
- **File**: `file.ts:line`
- **Risk**: HIGH - Injection or malformed data
- **Problem**: User input used without validation
- **Fix**: Add validation function before use
```

---

## Area 2: Injection Prevention

### Command Injection

**Safe Pattern**:
```typescript
// ✅ SAFE: Parameterized execution
import * as exec from '@actions/exec'

await exec.exec('bridge-cli', [
  '--stage', 'polaris',
  '--input', inputFilePath
])
```

**Dangerous Pattern**:
```typescript
// ❌ DANGEROUS: String concatenation
exec(`bridge-cli --stage ${userInput}`)
exec(`command ${param1} ${param2}`)
```

### Path Traversal

**Safe Pattern**:
```typescript
// ✅ SAFE: Path validation
import * as path from 'path'

const basePath = '/safe/directory'
const userPath = path.normalize(userInput)
const fullPath = path.join(basePath, userPath)

// Validate path stays within base directory
if (!fullPath.startsWith(basePath)) {
  throw new Error('Invalid path: directory traversal detected')
}

fs.readFile(fullPath)
```

**Dangerous Pattern**:
```typescript
// ❌ DANGEROUS: Direct user path
fs.readFile(userProvidedPath)

// ❌ DANGEROUS: String concatenation
fs.readFile(`/base/path/${userInput}`)
```

### SQL Injection (if applicable)

**Safe Pattern**:
```typescript
// ✅ SAFE: Parameterized query
db.query('SELECT * FROM users WHERE id = ?', [userId])
```

**Dangerous Pattern**:
```typescript
// ❌ DANGEROUS: String concatenation
db.query(`SELECT * FROM users WHERE id = ${userId}`)
```

### Search Patterns

```bash
# Find potential command injection
grep -rn "exec(\`" src/
grep -rn 'exec("' src/
grep -rn "spawn(\`" src/

# Find potential path traversal
grep -rn "fs.readFile" src/ | grep -v "path.join"
grep -rn "fs.writeFile" src/ | grep -v "path.join"

# Find SQL concatenation (if applicable)
grep -rn "query(\`" src/
grep -rn "query(\"" src/ | grep -v "?"
```

---

## Area 3: Unsafe Operations

### Never Use These

**Extremely Dangerous**:
```typescript
// ❌ NEVER USE
eval(userInput)
new Function(userInput)()
vm.runInNewContext(userInput)
```

**Avoid in Production**:
```typescript
// ⚠️ AVOID
element.innerHTML = userContent
dangerouslySetInnerHTML={{__html: userContent}}
document.write(userContent)
```

### Search Patterns

```bash
# Find eval usage
grep -rn "eval(" src/

# Find Function constructor
grep -rn "new Function" src/

# Find innerHTML
grep -rn "innerHTML" src/

# Find dangerous HTML injection
grep -rn "dangerouslySetInnerHTML" src/
grep -rn "document.write" src/
```

### Report Items

```markdown
### Unsafe Operations Found

**Issue**: Use of eval() detected
- **File**: `file.ts:line`
- **Risk**: CRITICAL - Arbitrary code execution
- **Problem**: eval() allows execution of arbitrary code
- **Fix**: Remove eval() and use safe alternatives
```

---

## Area 4: Secret Handling

### Good Patterns

**Use GitHub Secrets** (inputs.ts):
```typescript
// ✅ SAFE: Secrets from environment
export const POLARIS_ACCESS_TOKEN = core.getInput(
  constants.POLARIS_ACCESS_TOKEN_KEY
)

// Token automatically masked by GitHub Actions
core.setSecret(POLARIS_ACCESS_TOKEN)
```

**Never Log Secrets**:
```typescript
// ✅ SAFE: Log without exposing token
info(`Connecting to ${serverUrl}`)

// ❌ DANGEROUS: Token in logs
info(`Using token ${accessToken}`)
```

### Bad Patterns to Find

```typescript
// ❌ NEVER: Hardcoded credentials
const apiKey = 'sk_live_abc123...'
const password = 'admin123'
const token = 'ghp_abc123...'

// ❌ NEVER: Secrets in error messages
throw new Error(`Authentication failed with token ${token}`)

// ❌ NEVER: Secrets in version control
const config = {
  apiKey: 'secret-key-123'
}
```

### Search Patterns

```bash
# Find potential hardcoded secrets
grep -r "password\s*=\s*['\"]" src/
grep -r "api[-_]key\s*=\s*['\"]" src/
grep -r "secret\s*=\s*['\"]" src/
grep -r "token\s*=\s*['\"]" src/
grep -r "apiKey\s*:\s*['\"]" src/

# Find secrets in logs
grep -rn "info.*TOKEN" src/
grep -rn "debug.*PASSWORD" src/
grep -rn "console.log.*SECRET" src/
```

### Secret Files to Check

```bash
# Files that should not contain secrets
.env
config.json
credentials.json
secrets.json
.npmrc (with tokens)
```

### Report Items

```markdown
### Secret Handling Issues

**Issue**: Hardcoded API key found
- **File**: `file.ts:line`
- **Risk**: CRITICAL - Credential exposure
- **Problem**: API key hardcoded in source code
- **Fix**: Move to GitHub Secrets, use core.getInput()
```

---

## Area 5: Data Sanitization

### File Path Sanitization

```typescript
// ✅ SAFE: Normalize and validate
function sanitizePath(userPath: string, baseDir: string): string {
  // Normalize path (remove .., ./, etc.)
  const normalized = path.normalize(userPath)

  // Remove leading path traversal attempts
  const safe = normalized.replace(/^(\.\.(\/|\\|$))+/, '')

  // Join with base directory
  const fullPath = path.join(baseDir, safe)

  // Validate result is within base directory
  if (!fullPath.startsWith(baseDir)) {
    throw new Error('Path traversal attempt detected')
  }

  return fullPath
}
```

### URL Validation

```typescript
// ✅ SAFE: Validate URL
function isValidUrl(url: string): boolean {
  try {
    const parsed = new URL(url)
    // Enforce HTTPS
    return parsed.protocol === 'https:'
  } catch {
    return false
  }
}

// Use it
if (!isValidUrl(userUrl)) {
  throw new Error('Invalid URL')
}
```

### File Name Sanitization

```typescript
// ✅ SAFE: Sanitize file name
function sanitizeFileName(fileName: string): string {
  return fileName
    .replace(/[^a-zA-Z0-9.-]/g, '_') // Only allow safe chars
    .replace(/\.{2,}/g, '.') // Prevent multiple dots
    .slice(0, 255) // Limit length
}
```

### Report Items

```markdown
### Data Sanitization Issues

**Issue**: URL not validated
- **File**: `file.ts:line`
- **Risk**: MEDIUM - Malicious URL usage
- **Problem**: User-provided URL used without validation
- **Fix**: Validate URL format and protocol (HTTPS only)
```

---

## Area 6: Authentication & Authorization

### Token Storage

**Safe Patterns**:
```typescript
// ✅ SAFE: Token from environment
const token = core.getInput('github_token')
core.setSecret(token)

// ✅ SAFE: Token in memory only
const octokit = github.getOctokit(token)
```

**Dangerous Patterns**:
```typescript
// ❌ DANGEROUS: Token in localStorage
localStorage.setItem('token', token)

// ❌ DANGEROUS: Token in URL
fetch(`/api?token=${token}`)

// ❌ DANGEROUS: Token in logs
console.log(`Using token: ${token}`)
```

### Token Transmission

**Requirements**:
- Always use HTTPS for token transmission
- Use Authorization header, not query params
- Never log tokens
- Mask tokens in GitHub Actions with `core.setSecret()`

### Report Items

```markdown
### Authentication Issues

**Issue**: Token in query parameter
- **File**: `file.ts:line`
- **Risk**: HIGH - Token exposure in logs/URLs
- **Problem**: Token passed in URL query parameter
- **Fix**: Use Authorization header instead
```

---

## Area 7: Network Security

### HTTPS Enforcement

**Safe Pattern**:
```typescript
// ✅ SAFE: Validate HTTPS
function validateServerUrl(url: string): boolean {
  const parsed = new URL(url)
  if (parsed.protocol !== 'https:') {
    throw new Error('Server URL must use HTTPS')
  }
  return true
}
```

### SSL/TLS Configuration

**From ssl-utils.ts**:
```typescript
// ✅ SAFE: SSL verification enabled by default
export function configureSsl() {
  // Only disable SSL verification in air-gap mode
  if (inputs.NETWORK_AIRGAP === 'true') {
    process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0'
  } else {
    // Use proper SSL verification
    delete process.env.NODE_TLS_REJECT_UNAUTHORIZED
  }
}
```

### Proxy Configuration

**Safe Pattern**:
```typescript
// ✅ SAFE: Proxy from environment
const httpProxy = process.env.HTTP_PROXY || process.env.http_proxy
const httpsProxy = process.env.HTTPS_PROXY || process.env.https_proxy
```

### Report Items

```markdown
### Network Security Issues

**Issue**: SSL verification globally disabled
- **File**: `file.ts:line`
- **Risk**: HIGH - Man-in-the-middle attacks
- **Problem**: NODE_TLS_REJECT_UNAUTHORIZED disabled unconditionally
- **Fix**: Only disable in specific air-gap scenarios
```

---

## Area 8: File System Security

### Temp File Cleanup

**Good Pattern** (main.ts):
```typescript
let tempDir: string

try {
  // Create temp directory
  tempDir = fs.mkdtempSync(path.join(os.tmpdir(), 'action_temp_'))

  // Use temp directory for operations
  await performOperations(tempDir)
} finally {
  // Always cleanup, even if error
  if (tempDir && fs.existsSync(tempDir)) {
    fs.rmSync(tempDir, {recursive: true, force: true})
  }
}
```

### File Permissions

```typescript
// ✅ SAFE: Restrictive permissions
fs.writeFileSync(secretFile, data, {mode: 0o600}) // Owner only

// ❌ DANGEROUS: World-readable secrets
fs.writeFileSync(secretFile, data, {mode: 0o644})
```

### Directory Traversal Prevention

```typescript
// ✅ SAFE: Validate paths
const safePath = path.join(baseDir, path.normalize(userPath))
if (!safePath.startsWith(baseDir)) {
  throw new Error('Path traversal detected')
}
```

### Report Items

```markdown
### File System Security Issues

**Issue**: Missing temp file cleanup
- **File**: `file.ts:line`
- **Risk**: MEDIUM - Temp file leakage
- **Problem**: Temp files not cleaned up in error path
- **Fix**: Use finally block for cleanup
```

---

## Area 9: Dependency Security

### Check for Vulnerabilities

```bash
# Check for known vulnerabilities
npm audit

# Show detailed vulnerability report
npm audit --json

# Auto-fix vulnerabilities
npm audit fix

# Check for outdated packages
npm outdated

# Check specific package
npm audit --package=<package-name>
```

### Review Dependencies

**Check**:
- Are all dependencies necessary?
- Are versions up to date?
- Are there known vulnerabilities?
- Are dependencies from trusted sources?

### Report Items

```markdown
### Dependency Security Issues

**Summary**:
- Critical vulnerabilities: [count]
- High vulnerabilities: [count]
- Medium vulnerabilities: [count]
- Low vulnerabilities: [count]

**Critical Issues**:
1. [@package/name](version): [vulnerability description]
   - **Risk**: [Impact]
   - **Fix**: Update to version [X.Y.Z]
```

---

## Area 10: Logging & Monitoring

### Secure Logging

**Safe Patterns**:
```typescript
// ✅ SAFE: Log without secrets
import {info, warning, error} from '@actions/core'

info('Starting Bridge CLI execution')
info(`Scanning with products: ${productList}`)

// Secrets automatically masked if registered with core.setSecret()
core.setSecret(accessToken)
info(`Token: ${accessToken}`) // Will show as ***
```

**Dangerous Patterns**:
```typescript
// ❌ DANGEROUS: Secrets in logs
console.log(`Access token: ${accessToken}`)
console.log(`Password: ${password}`)

// ❌ DANGEROUS: Sensitive data in error messages
throw new Error(`Failed to authenticate with token ${token}`)

// ❌ DANGEROUS: Full request/response bodies
console.log(`Response: ${JSON.stringify(response)}`)
```

### Report Items

```markdown
### Logging Security Issues

**Issue**: Secret exposed in log statement
- **File**: `file.ts:line`
- **Risk**: CRITICAL - Secret exposure in logs
- **Problem**: Access token logged to console
- **Fix**: Remove secret from log or use core.setSecret()
```

---

## OWASP Top 10 Checklist

Run through this checklist for comprehensive security review:

### 1. Injection
- [ ] Command injection prevented (parameterized commands)
- [ ] Path traversal prevented (path validation)
- [ ] No string concatenation in system commands
- [ ] SQL injection prevented (if using DB)

### 2. Broken Authentication
- [ ] Secrets from environment/GitHub Secrets
- [ ] No hardcoded credentials
- [ ] Tokens properly masked
- [ ] Token expiration handled

### 3. Sensitive Data Exposure
- [ ] Secrets not in logs
- [ ] Secrets not in error messages
- [ ] SSL/TLS enforced
- [ ] Sensitive files excluded from git

### 4. XML External Entities (if parsing XML)
- [ ] XML parser configured securely
- [ ] External entity expansion disabled

### 5. Broken Access Control
- [ ] File access validated
- [ ] Path traversal prevented
- [ ] Directory boundaries enforced

### 6. Security Misconfiguration
- [ ] Defaults are secure
- [ ] Development configs not in production
- [ ] Error messages don't expose internals
- [ ] Security headers set

### 7. Cross-Site Scripting (if generating HTML)
- [ ] User input sanitized
- [ ] No innerHTML with user data
- [ ] Content Security Policy set

### 8. Insecure Deserialization
- [ ] JSON.parse() validated
- [ ] Type checking after parsing
- [ ] No eval() on parsed data

### 9. Using Components with Known Vulnerabilities
- [ ] Dependencies up to date
- [ ] No known vulnerable versions
- [ ] Regular npm audit runs

### 10. Insufficient Logging & Monitoring
- [ ] Security events logged
- [ ] Errors properly logged
- [ ] Logs don't contain secrets

---

## Security Scanning Process

### Step 1: Automated Scans

```bash
# Dependency vulnerabilities
npm audit

# TypeScript compilation (finds some issues)
npm run build

# Linting (finds some security issues)
npm run lint
```

### Step 2: Pattern-Based Search

```bash
# Secrets
grep -r "password.*=.*['\"]" src/
grep -r "api[-_]key.*=.*['\"]" src/
grep -r "secret.*=.*['\"]" src/

# Unsafe operations
grep -r "eval(" src/
grep -r "new Function" src/
grep -r "innerHTML" src/

# Command execution
grep -r "exec(" src/
grep -r "spawn(" src/

# File operations
grep -r "fs.readFile" src/
grep -r "fs.writeFile" src/
```

### Step 3: Manual Code Review

For each security area:
1. Check patterns in code
2. Validate against best practices
3. Document findings
4. Prioritize by risk

### Step 4: Generate Report

---

## Comprehensive Security Report Format

```markdown
# Security Review Report

**Date**: [Date]
**Reviewer**: Security Review Skill
**Scope**: Full security audit

## Executive Summary

### Overall Security Posture: [SECURE/VULNERABLE/CRITICAL]

### Critical Issues: [Count]
### High Priority: [Count]
### Medium Priority: [Count]
### Low Priority: [Count]

## Critical Issues (Fix Immediately)

### 1. Hardcoded Secret Found
- **File**: `file.ts:line`
- **Category**: Secret Handling
- **Risk**: CRITICAL - Credential exposure
- **Problem**: API key hardcoded in source code
- **Impact**: If repository is leaked, credentials are compromised
- **Fix**:
  1. Remove hardcoded secret
  2. Add to GitHub Secrets
  3. Use core.getInput() to retrieve
  4. Rotate the exposed secret

### 2. Command Injection Vulnerability
- **File**: `file.ts:line`
- **Category**: Injection
- **Risk**: CRITICAL - Arbitrary command execution
- **Problem**: User input concatenated into command string
- **Impact**: Attacker can execute arbitrary commands
- **Fix**:
  1. Use parameterized execution: `exec.exec('cmd', [arg1, arg2])`
  2. Validate all inputs before use

## High Priority Issues

### 1. Path Traversal Vulnerability
- **File**: `file.ts:line`
- **Category**: File System Security
- **Risk**: HIGH - Unauthorized file access
- **Problem**: User-provided path not validated
- **Impact**: Attacker can read/write arbitrary files
- **Fix**:
  1. Normalize path: `path.normalize()`
  2. Validate against base directory
  3. Reject if outside allowed paths

## Medium Priority Issues

[Similar format]

## Low Priority Issues

[Similar format]

## OWASP Top 10 Assessment

| Category | Status | Issues Found |
|----------|--------|--------------|
| Injection | ✅/⚠️/❌ | [count] |
| Broken Authentication | ✅/⚠️/❌ | [count] |
| Sensitive Data Exposure | ✅/⚠️/❌ | [count] |
| XML External Entities | ✅/⚠️/❌ | [count] |
| Broken Access Control | ✅/⚠️/❌ | [count] |
| Security Misconfiguration | ✅/⚠️/❌ | [count] |
| Cross-Site Scripting | ✅/⚠️/❌ | [count] |
| Insecure Deserialization | ✅/⚠️/❌ | [count] |
| Components with Vulnerabilities | ✅/⚠️/❌ | [count] |
| Insufficient Logging | ✅/⚠️/❌ | [count] |

## Dependency Vulnerabilities

```bash
npm audit summary:
Critical: [count]
High: [count]
Medium: [count]
Low: [count]
```

**Action Required**:
1. [Package](version): [Vulnerability]
   - Update to: [version]

## Security Best Practices Found

✅ Secrets from GitHub Secrets (inputs.ts)
✅ Parameterized command execution (@actions/exec)
✅ Path normalization (path.join, path.normalize)
✅ Temp file cleanup (main.ts finally block)
✅ SSL configuration (ssl-utils.ts)
✅ Input validation (validators.ts)

## Security Anti-Patterns Found

❌ [Anti-pattern] at `file.ts:line`

## Recommendations

### Immediate Actions (This Week)
1. Remove all hardcoded secrets
2. Fix command injection vulnerabilities
3. Add input validation to all external inputs
4. Rotate any exposed credentials

### Short-term Actions (This Month)
1. Review all file operations for path traversal
2. Add security headers
3. Implement rate limiting if applicable
4. Set up automated security scanning in CI/CD

### Long-term Actions (This Quarter)
1. Regular dependency audits (weekly)
2. Security training for team
3. Penetration testing
4. Implement security monitoring

## Compliance Considerations

- **GDPR**: Handle user data appropriately
- **SOC 2**: Audit logging, access controls
- **PCI DSS**: If handling payment data
- **HIPAA**: If handling health data

## Next Steps

1. Address critical issues immediately
2. Create GitHub issues for high/medium priority items
3. Schedule security review after fixes
4. Set up automated security scanning
```

---

## Best Practices

### Review Principles
- **Risk-based**: Prioritize by impact and likelihood
- **Comprehensive**: Cover all OWASP Top 10
- **Actionable**: Provide specific fix recommendations
- **Verifiable**: Include file:line references

### When to Run
- Before every release
- After adding new features
- When dependencies change
- Weekly as part of security hygiene
- After security incidents

### Tools to Use
- `npm audit` for dependency scanning
- `grep` for pattern-based searching
- Manual code review for logic flaws
- Static analysis tools (ESLint security plugins)

---

## Example Usage

**User**: "Review code security"

**Response**:
1. Run npm audit
2. Search for dangerous patterns
3. Review each security area
4. Generate comprehensive report
5. Prioritize by risk level
6. Provide specific fixes
