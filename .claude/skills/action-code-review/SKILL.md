---
name: action-code-review
description: Comprehensive code review skill that analyzes code quality across linting/formatting, type safety, error handling, refactoring opportunities, testing quality, design patterns, and documentation. Use for thorough code quality analysis. For security-specific reviews, use the security-review skill instead.
---

# Code Review

Reviews code changes against established patterns and coding standards. Always read changed files before producing findings. Never flag style that matches existing code.

## Usage

Run this skill when the user requests:
- "Review this code"
- "Check code quality"
- "Review code standards"
- "Check my changes"
- "What did I break?"
- "Review design patterns"
- "Check test coverage"

NOTE: For security-focused reviews, use the `security-review` skill instead.

## How to Use

When conducting a code review:

1. **Read all changed files** (git diff or user-provided files)
2. **Run each checklist** section below against the changes
3. **Report findings** as: `file:line — [SEVERITY] — problem — correct pattern`
4. **Use severity tags**: `BREAK`, `COMPAT`, `STANDARD`, `TEST`

---

## Severity Reference

| Tag | Meaning | Examples |
|---|---|---|
| `BREAK` | Will fail at runtime or silently produce wrong output | Missing required params, type errors, undefined variables |
| `COMPAT` | Backward compatibility risk for existing workflows | Removing/renaming inputs, changing validation logic |
| `STANDARD` | Deviates from project convention, inconsistency risk | Wrong naming convention, missing JSDoc, style violations |
| `TEST` | Missing or incorrect test coverage | No tests for new code, tests don't reset state |

---

## Checklist 1 — Input Handling Standards

**Rule: Every new input must be exported from `inputs.ts` using `core.getInput()`.**

**Context:** All GitHub Action inputs are centralized in `src/blackduck-security-action/inputs.ts` for consistency and to handle deprecated parameter names.

**Checks:**
- [ ] New input exported from `inputs.ts` — NOT read inline with `core.getInput()` in other files
- [ ] Input key constant in `application-constants.ts` follows `{PRODUCT}_{PARAM}_KEY` naming
- [ ] No hardcoded string keys — all keys go through constants
- [ ] Deprecated params use proper fallback mechanism (old key checked if new key empty)
- [ ] Boolean inputs use `core.getBooleanInput()`, not `core.getInput()` with string comparison

**Remediation pattern:**
```typescript
// ❌ WRONG — reading inline
const serverUrl = core.getInput('polaris_server_url')

// ✅ CORRECT — export from inputs.ts
// In application-constants.ts
export const POLARIS_SERVER_URL_KEY = 'polaris_server_url'

// In inputs.ts
export const POLARIS_SERVER_URL = core.getInput(constants.POLARIS_SERVER_URL_KEY)

// In other files
import * as inputs from './inputs'
const serverUrl = inputs.POLARIS_SERVER_URL
```

**BREAK risk:** Reading inputs directly breaks the single source of truth and skips deprecation handling.

---

## Checklist 2 — Type Safety Standards

**Rule: All functions must have explicit parameter types and return types. No implicit `any`.**

**Context:** The codebase uses strict TypeScript with `noImplicitAny: true` in `tsconfig.json`.

**Checks:**
- [ ] All function parameters have explicit types — no implicit `any`
- [ ] All functions have explicit return types (including `void`)
- [ ] Async functions return `Promise<T>`, not bare `T`
- [ ] No `as any` type assertions without justification
- [ ] Type guards used instead of unsafe type assertions
- [ ] Null/undefined handled explicitly with optional chaining (`?.`) or nullish coalescing (`??`)
- [ ] Interface properties documented with JSDoc

**Remediation pattern:**
```typescript
// ❌ WRONG — implicit types
function process(input) {
  return input.split(',')
}

// ❌ WRONG — no return type
function process(input: string) {
  return input.split(',')
}

// ✅ CORRECT — explicit types
function process(input: string): string[] {
  return input.split(',')
}

// ✅ CORRECT — async with Promise
async function fetchData(url: string): Promise<string> {
  const response = await fetch(url)
  return response.text()
}

// ✅ CORRECT — null safety
function process(input: string | null): string {
  return input?.trim() ?? ''
}
```

**BREAK risk:** Implicit `any` bypasses type checking and can cause runtime errors.

---

## Checklist 3 — Error Handling Standards

**Rule: All thrown errors must be descriptive and preserve context. Async operations must handle errors.**

**Context:** Errors propagate to `main.ts` catch block and are displayed in GitHub Actions logs.

**Checks:**
- [ ] Every `throw new Error()` includes descriptive message and context
- [ ] Async operations wrapped in try-catch
- [ ] Errors re-thrown with added context, not swallowed
- [ ] Validation errors collected in arrays, not fail-fast
- [ ] No empty catch blocks
- [ ] Error messages reference parameter names from `application-constants.ts`
- [ ] No sensitive data (tokens, passwords) in error messages

**Remediation pattern:**
```typescript
// ❌ WRONG — swallowing error
try {
  await operation()
} catch (error) {
  // Silent failure
}

// ❌ WRONG — no context
try {
  await operation()
} catch (error) {
  throw error
}

// ✅ CORRECT — add context and re-throw
try {
  await downloadBridgeCli(url)
} catch (error) {
  throw new Error(`Failed to download Bridge CLI from ${url}: ${error}`)
}

// ✅ CORRECT — collect validation errors
function validateInputs(): string[] {
  const errors: string[] = []
  if (!inputs.SERVER_URL) {
    errors.push('server_url is required')
  }
  if (!inputs.ACCESS_TOKEN) {
    errors.push('access_token is required')
  }
  return errors
}
```

**STANDARD risk:** Poor error handling makes debugging difficult.

---

## Checklist 4 — Logging Standards

**Rule: Use `@actions/core` logging functions. Debug info → `core.debug()`, user messages → `core.info()`.**

**Context:** GitHub Actions provides structured logging via `@actions/core`.

**Checks:**
- [ ] Internal state/variables logged with `core.debug()` — NOT `console.log()`
- [ ] User-visible progress/status logged with `core.info()`
- [ ] Warnings logged with `core.warning()`
- [ ] Errors logged with `core.error()` (only from `main.ts` catch block)
- [ ] No `console.log()`, `console.error()`, `console.warn()` in production code
- [ ] No secrets or tokens in any log level
- [ ] `core.setSecret()` called for credentials before any operations

**Remediation pattern:**
```typescript
// ❌ WRONG — console.log
console.log(`Processing product: ${productName}`)
console.log(`Token: ${accessToken}`)

// ✅ CORRECT — structured logging
core.info(`Processing product: ${productName}`)
core.debug(`Bridge CLI path: ${cliPath}`)
core.setSecret(accessToken)
core.info('Access token configured')
```

**STANDARD risk:** Inconsistent logging makes troubleshooting harder.

---

## Checklist 5 — Constants and Magic Values

**Rule: All string literals, numbers, and configuration values must be constants in `application-constants.ts`.**

**Context:** Centralized constants prevent typos and make updates easier.

**Checks:**
- [ ] No magic numbers — extract to named constants
- [ ] No hardcoded strings (URLs, keys, messages) — use constants
- [ ] Exit codes defined in constants with descriptive names
- [ ] File names, stage names, parameter keys all in `application-constants.ts`
- [ ] Version thresholds defined as named constants (not inline `"2.0.0"`)

**Remediation pattern:**
```typescript
// ❌ WRONG — magic values
if (exitCode === 8) {
  // What does 8 mean?
}
const inputFile = 'polaris_input.json'

// ✅ CORRECT — named constants
// In application-constants.ts
export const BRIDGE_EXIT_CODE_POLICY_VIOLATION = 8
export const POLARIS_INPUT_FILE = 'polaris_input.json'

// In code
if (exitCode === constants.BRIDGE_EXIT_CODE_POLICY_VIOLATION) {
  // Clear intent
}
const inputFile = constants.POLARIS_INPUT_FILE
```

**STANDARD risk:** Magic values are hard to maintain and prone to typos.

---

## Checklist 6 — Test Coverage Standards

**Rule: Every new function and every branch must be tested. Tests must be independent.**

**Context:** Uses Jest for unit tests and contract tests. Tests in `test/unit/` and `test/contract/`.

**Checks:**
- [ ] New functions have unit tests verifying behavior
- [ ] Both success and failure paths tested
- [ ] All conditional branches (if/else, switch) have test cases
- [ ] Tests use Arrange-Act-Assert pattern
- [ ] Tests are independent — no shared state between tests
- [ ] `afterEach` resets ALL mocked inputs touched in any test in the context
- [ ] Mock objects restored with `jest.restoreAllMocks()` in `afterEach`
- [ ] Generated files cleaned up in `afterEach`

**Remediation pattern:**
```typescript
// ✅ CORRECT — test structure
describe('validatePolarisInputs', () => {
  afterEach(() => {
    // Clean up ALL inputs touched in ANY test
    Object.defineProperty(inputs, 'POLARIS_SERVER_URL', { value: '' })
    Object.defineProperty(inputs, 'POLARIS_ACCESS_TOKEN', { value: '' })
    jest.restoreAllMocks()
  })

  it('should return empty array when all required inputs provided', () => {
    // Arrange
    Object.defineProperty(inputs, 'POLARIS_SERVER_URL', { value: 'https://polaris.example.com' })
    Object.defineProperty(inputs, 'POLARIS_ACCESS_TOKEN', { value: 'token123' })

    // Act
    const errors = validatePolarisInputs()

    // Assert
    expect(errors).toEqual([])
  })

  it('should return error when server URL missing', () => {
    // Arrange
    Object.defineProperty(inputs, 'POLARIS_SERVER_URL', { value: '' })
    Object.defineProperty(inputs, 'POLARIS_ACCESS_TOKEN', { value: 'token123' })

    // Act
    const errors = validatePolarisInputs()

    // Assert
    expect(errors.length).toBeGreaterThan(0)
    expect(errors).toContain(expect.stringContaining('polaris_server_url'))
  })
})
```

**TEST risk:** Missing tests allow bugs to slip through.

---

## Checklist 7 — Design Pattern Adherence

**Rule: Follow established patterns. Don't invent new structures when existing patterns work.**

**Context:** Codebase uses Factory, Builder, Strategy, and Template Method patterns.

**Checks:**
- [ ] New product follows existing product pattern (look at Polaris, Coverity)
- [ ] Input data models extend `Common` interface
- [ ] Validators return `string[]` of errors (empty if valid)
- [ ] Command builders in `tools-parameter.ts` follow same structure
- [ ] Services implement appropriate interface (`IGithubClientService`)
- [ ] No duplicate code — extract to utilities if pattern repeats >2 times

**Remediation pattern:**
```typescript
// ❌ WRONG — new pattern for same problem
function validateMyProduct(): boolean {
  if (!input) return false
  return true
}

// ✅ CORRECT — follow existing pattern
function validateMyProductInputs(): string[] {
  const errors: string[] = []

  if (!inputs.MYPRODUCT_SERVER_URL) {
    errors.push('myproduct_server_url is required')
  }

  return errors
}
```

**STANDARD risk:** Inconsistent patterns make codebase harder to understand.

---

## Checklist 8 — Documentation Standards

**Rule: All public functions must have JSDoc. Complex logic needs inline comments.**

**Context:** JSDoc provides IDE autocomplete and generates documentation.

**Checks:**
- [ ] All exported functions have JSDoc with description
- [ ] JSDoc includes `@param` for all parameters
- [ ] JSDoc includes `@returns` for return value
- [ ] JSDoc includes `@throws` if function can throw
- [ ] Complex logic has inline comments explaining WHY, not WHAT
- [ ] Interfaces have JSDoc describing purpose
- [ ] Interface properties documented where non-obvious

**Remediation pattern:**
```typescript
// ❌ WRONG — no documentation
export function validateInputs(): string[] {
  // ...
}

// ✅ CORRECT — comprehensive JSDoc
/**
 * Validates Polaris input parameters
 *
 * Checks that all required parameters for Polaris scanning are provided
 * and valid. Only performs validation if Polaris is enabled.
 *
 * @returns Array of error messages (empty if all inputs valid)
 *
 * @example
 * ```typescript
 * const errors = validatePolarisInputs()
 * if (errors.length > 0) {
 *   throw new Error(errors.join(', '))
 * }
 * ```
 */
export function validatePolarisInputs(): string[] {
  const errors: string[] = []
  // Validation logic
  return errors
}
```

**STANDARD risk:** Missing documentation makes code hard to use and maintain.

---

## Checklist 9 — File Organization Standards

**Rule: Files belong in specific directories by responsibility. No 1000+ line files.**

**Context:** Code organized by layer: constants, models, validators, services, utilities.

**Checks:**
- [ ] Constants in `application-constants.ts` — not scattered across files
- [ ] Input reading in `inputs.ts` — not inline in other files
- [ ] Data models in `input-data/` directory
- [ ] Validators in `validators.ts`
- [ ] Services in `service/` directory
- [ ] No files >500 lines (refactor if needed)
- [ ] Related functions grouped together

**Remediation pattern:**
```typescript
// ❌ WRONG — constants scattered
// In validators.ts
const SERVER_URL_KEY = 'polaris_server_url'

// In tools-parameter.ts
const SERVER_URL_KEY = 'polaris_server_url'

// ✅ CORRECT — centralized
// In application-constants.ts
export const POLARIS_SERVER_URL_KEY = 'polaris_server_url'

// In validators.ts and tools-parameter.ts
import * as constants from './application-constants'
const key = constants.POLARIS_SERVER_URL_KEY
```

**STANDARD risk:** Poor organization makes code hard to find and maintain.

---

## Checklist 10 — Backward Compatibility

**Rule: Don't break existing workflows. Support deprecated parameters with warnings.**

**Context:** Workflows in production use action inputs. Breaking changes cause silent failures.

**Checks:**
- [ ] No input parameters renamed without deprecation support
- [ ] No required fields added that were previously optional
- [ ] No validation logic changed to reject previously valid inputs
- [ ] Deprecated parameters still work with warning in logs
- [ ] `action.yml` inputs match constants in `application-constants.ts`
- [ ] Default values not changed without major version bump

**Remediation pattern:**
```typescript
// ❌ WRONG — breaking change
export const NEW_PARAM = core.getInput('new_param_name')

// ✅ CORRECT — backward compatible
export const NEW_PARAM = core.getInput('new_param_name') ||
                          core.getInput('old_param_name') // Deprecated fallback
if (core.getInput('old_param_name')) {
  core.warning('old_param_name is deprecated, use new_param_name instead')
}
```

**COMPAT risk:** Breaking changes silently fail workflows without errors.

---

## Code Review Report Format

```markdown
# Code Review Report

**Date**: [Date]
**Reviewer**: Code Review Skill
**Files Reviewed**: [count]

## Summary

- **BREAK Issues**: [count]
- **COMPAT Issues**: [count]
- **STANDARD Issues**: [count]
- **TEST Issues**: [count]

**Overall**: ✅ APPROVED / ⚠️ NEEDS WORK / ❌ REJECTED

---

## Critical Issues (Must Fix)

### [BREAK] file.ts:line — Missing return type
**Problem**: Function `validateInputs` has no return type annotation
**Impact**: Type checking bypassed, could return unexpected types
**Fix**:
```typescript
// Current
function validateInputs() {
  return errors
}

// Fixed
function validateInputs(): string[] {
  return errors
}
```

---

## High Priority Issues (Should Fix)

### [COMPAT] file.ts:line — Input renamed without deprecation
**Problem**: `old_param` renamed to `new_param` breaks existing workflows
**Impact**: Existing workflows fail silently
**Fix**:
```typescript
export const NEW_PARAM = core.getInput('new_param') ||
                          core.getInput('old_param')
```

---

## Standard Issues (Nice to Fix)

### [STANDARD] file.ts:line — Magic number
**Problem**: Hardcoded `8` without explanation
**Impact**: Hard to maintain, unclear intent
**Fix**:
```typescript
// In application-constants.ts
export const POLICY_VIOLATION_EXIT_CODE = 8

// In code
if (exitCode === constants.POLICY_VIOLATION_EXIT_CODE) {
```

---

## Test Coverage Issues

### [TEST] validators.ts — Missing test for error path
**Problem**: No test for missing required parameter
**Impact**: Regression risk
**Fix**: Add test case for validation failure

---

## Recommendations

1. Fix all BREAK issues before merging
2. Address COMPAT issues to prevent breaking workflows
3. Clean up STANDARD issues for consistency
4. Add missing tests before release
```

---

## Quick Review Checklist

Before approving code:

- [ ] All functions have explicit types
- [ ] All errors have descriptive messages
- [ ] No console.log (use core.info/debug)
- [ ] Constants defined in application-constants.ts
- [ ] Tests cover new code and branches
- [ ] JSDoc on all exported functions
- [ ] No hardcoded strings or magic numbers
- [ ] Backward compatible (no breaking changes)
- [ ] Secrets masked with core.setSecret()
- [ ] Clean up in afterEach for all tests

---

## Best Practices

### Review Principles
- **Be objective**: Focus on code, not author
- **Be specific**: Provide file:line references
- **Be constructive**: Suggest solutions, not just problems
- **Prioritize**: BREAK > COMPAT > STANDARD > TEST

### When to Run
- Before merging pull requests
- After significant changes
- Before releases
- During code pairing sessions

### Tools to Use
- `npm run lint` for linting
- `npm run format-check` for formatting
- `npm test` for test coverage
- `npm run build` for type checking
