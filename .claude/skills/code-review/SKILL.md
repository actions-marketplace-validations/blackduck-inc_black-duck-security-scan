---
name: code-review
description: Comprehensive code review skill that analyzes code quality across linting/formatting, type safety, error handling, refactoring opportunities, testing quality, design patterns, and documentation. Use for thorough code quality analysis. For security-specific reviews, use the security-review skill instead.
---

# Code Review

Performs comprehensive code quality review of the Black Duck Security Scan codebase, covering style, types, errors, patterns, tests, and documentation.

## Usage

Run this skill when the user requests:
- "Review this code"
- "Check code quality"
- "Review code standards"
- "Audit code quality"
- "Check for code smells"
- "Review design patterns"
- "Check test coverage"
- "Review documentation"

NOTE: For security-focused reviews, use the `security-review` skill instead.

## Review Areas

This skill covers 7 key areas:

1. **Linting & Formatting** - Code style and formatting consistency
2. **Type Safety** - TypeScript strict typing compliance
3. **Error Handling** - Proper error handling patterns
4. **Refactoring Opportunities** - Code smells and improvement areas
5. **Testing Quality** - Test coverage and quality
6. **Design Patterns** - Pattern usage and violations
7. **Documentation** - Code documentation coverage

---

## Area 1: Linting & Formatting

### What to Check

#### ESLint Configuration
- Check `.eslintrc` or `eslint.config.mjs` for rules
- Run `npm run lint` to identify violations
- Review auto-fix suggestions with `npm run lint-fix`

#### Prettier Configuration
- Check `.prettierrc` or `package.json` for formatting rules
- Run `npm run format-check` to identify formatting issues
- Auto-fix with `npm run format`

#### Common Issues

**Formatting Issues**:
- ❌ Semicolons (should be removed)
- ❌ Double quotes (should be single quotes)
- ❌ Inconsistent indentation (should be 2 spaces)
- ❌ Trailing commas
- ❌ Unnecessary parentheses in arrow functions with single params

**Linting Issues**:
- ❌ Unused variables
- ❌ Console statements (use @actions/core instead)
- ❌ Prefer const over let
- ❌ Missing return types
- ❌ Complexity violations (functions too complex)

### Review Process

```bash
# Check formatting
npm run format-check

# Auto-fix formatting
npm run format

# Check linting
npm run lint

# Auto-fix linting issues
npm run lint-fix
```

### Report Format

```markdown
## Linting & Formatting

**Status**: ✅ PASS / ⚠️ ISSUES / ❌ FAIL

**Summary**:
- Formatting issues: [count]
- Linting errors: [count]
- Linting warnings: [count]

**Issues Found**:
1. [File:line] - [Issue description]
2. [File:line] - [Issue description]

**Fix Command**: `npm run format && npm run lint-fix`
```

---

## Area 2: Type Safety

### What to Check

#### TypeScript Configuration
Review `tsconfig.json` for strict mode settings:
```json
{
  "strict": true,
  "noImplicitAny": true,
  "strictNullChecks": true,
  "strictFunctionTypes": true,
  "strictPropertyInitialization": true
}
```

#### Common Type Safety Issues

**Implicit Any**:
```typescript
// ❌ BAD
function process(data) {
  return data.value
}

// ✅ GOOD
function process(data: InputData): string {
  return data.value
}
```

**Missing Return Types**:
```typescript
// ❌ BAD
function calculate(x: number, y: number) {
  return x + y
}

// ✅ GOOD
function calculate(x: number, y: number): number {
  return x + y
}
```

**Null Safety Issues**:
```typescript
// ❌ BAD
function process(input: string | null) {
  return input.toUpperCase() // Error if null
}

// ✅ GOOD
function process(input: string | null): string {
  if (input == null) {
    return ''
  }
  return input.toUpperCase()
}
```

**Unsafe Type Assertions**:
```typescript
// ❌ BAD
const data = response as MyType

// ✅ GOOD
function isMyType(obj: unknown): obj is MyType {
  return typeof obj === 'object' && obj !== null && 'field' in obj
}

if (!isMyType(response)) {
  throw new TypeError('Invalid response')
}
const data = response
```

### Review Process

```bash
# Type check without emitting
npx tsc --noEmit

# Build (includes type checking)
npm run build
```

### Analysis Steps

1. **Grep for implicit any**:
   - Search for functions without parameter types
   - Search for variables without explicit types
   - Check for `any` type usage

2. **Check return types**:
   - Ensure all functions have explicit return types
   - Verify async functions return `Promise<T>`
   - Check void returns are explicit

3. **Review null handling**:
   - Check for `?.` optional chaining usage
   - Check for `??` nullish coalescing
   - Verify null checks before operations

4. **Review type assertions**:
   - Flag `as` type assertions without validation
   - Look for type guards instead
   - Check for proper runtime validation

### Report Format

```markdown
## Type Safety

**Status**: ✅ PASS / ⚠️ ISSUES / ❌ FAIL

**Summary**:
- Implicit any: [count]
- Missing return types: [count]
- Null safety issues: [count]
- Unsafe assertions: [count]
- Type safety score: [X]%

**Critical Issues**:
1. [File:line] - [Issue description]
2. [File:line] - [Issue description]

**Recommendations**:
- Add explicit return types to all functions
- Replace `as` assertions with type guards
- Add null checks for optional parameters
```

---

## Area 3: Error Handling

### What to Check

#### Try-Catch Usage

**Good Patterns**:
```typescript
// ✅ Proper try-catch with context
try {
  await executeCommand()
} catch (error) {
  throw new Error(`Failed to execute command: ${error}`)
}

// ✅ Error transformation
try {
  const data = JSON.parse(input)
} catch (error) {
  throw new TypeError(`Invalid JSON input: ${error}`)
}

// ✅ Validation error collection
const errors: string[] = []
if (!field1) errors.push('field1 is required')
if (!field2) errors.push('field2 is required')
if (errors.length > 0) {
  throw new Error(errors.join(', '))
}
```

**Bad Patterns**:
```typescript
// ❌ Swallowing errors
try {
  await operation()
} catch (error) {
  // Silent failure
}

// ❌ Generic catch without re-throw
try {
  await operation()
} catch (error) {
  console.log('Error occurred')
  // Lost error context
}

// ❌ Catching without handling
try {
  await operation()
} catch (error) {
  throw error // No transformation or context
}

// ❌ Missing async error handling
async function process() {
  const result = await operation() // No try-catch
  return result
}
```

### Review Process

1. **Find all try-catch blocks**:
   ```bash
   grep -rn "try {" src/
   ```

2. **Check error handling patterns**:
   - Are errors transformed with context?
   - Are errors logged appropriately?
   - Are errors re-thrown or handled?
   - Is there validation error collection?

3. **Check async error handling**:
   - Are async operations wrapped in try-catch?
   - Are Promises rejected properly?
   - Is error propagation clear?

4. **Check validation patterns**:
   - Are validation errors collected?
   - Do validators return error arrays?
   - Are all validators called?

### Report Format

```markdown
## Error Handling

**Status**: ✅ GOOD / ⚠️ NEEDS IMPROVEMENT / ❌ CRITICAL

**Summary**:
- Try-catch blocks reviewed: [count]
- Issues found: [count]
- Critical issues: [count]

**Issues**:
1. [File:line] - [Issue description]
   - **Problem**: [What's wrong]
   - **Impact**: [Consequence]
   - **Fix**: [How to fix]

**Recommendations**:
- Add error context when re-throwing
- Use validation error collection pattern
- Wrap all async operations in try-catch
```

---

## Area 4: Refactoring Opportunities

### What to Check

#### Code Smells

**1. Large Functions** (>50 lines):
```bash
# Find large functions
grep -A 100 "function\|=>" src/**/*.ts | wc -l
```

**2. Deep Nesting** (>3 levels):
```typescript
// ❌ BAD: Deep nesting
if (condition1) {
  if (condition2) {
    if (condition3) {
      if (condition4) {
        // Too deep
      }
    }
  }
}

// ✅ GOOD: Early returns
if (!condition1) return
if (!condition2) return
if (!condition3) return
if (!condition4) return
// Main logic
```

**3. Code Duplication**:
- Similar code patterns across files
- Repeated validation logic
- Duplicated transformations

**4. Magic Numbers/Strings**:
```typescript
// ❌ BAD
if (exitCode === 8) {
  // What does 8 mean?
}

// ✅ GOOD
if (exitCode === constants.BRIDGE_EXIT_CODE_POLICY_VIOLATION) {
  // Clear intent
}
```

**5. Long Parameter Lists** (>4 parameters):
```typescript
// ❌ BAD
function create(name, url, token, type, enabled, timeout, retries) {
  // Too many params
}

// ✅ GOOD
interface CreateConfig {
  name: string
  url: string
  token: string
  type: string
  enabled: boolean
  timeout: number
  retries: number
}

function create(config: CreateConfig) {
  // Much better
}
```

**6. God Objects** (classes with too many responsibilities):
- Check for classes with >10 public methods
- Check for classes with >500 lines
- Look for mixed responsibilities

**7. Feature Envy** (method uses more of another class than its own):
```typescript
// ❌ BAD
class OrderProcessor {
  process(order: Order) {
    // Using Customer methods too much
    if (order.customer.hasDiscountEligibility()) {
      const discount = order.customer.calculateDiscount()
      const total = order.customer.applyDiscount(discount)
    }
  }
}

// ✅ GOOD: Move to Customer class
```

### Review Process

1. **Analyze file sizes**:
   ```bash
   wc -l src/**/*.ts | sort -nr
   ```

2. **Check function complexity**:
   - Use ESLint complexity rules
   - Identify functions >50 lines
   - Look for nested conditionals

3. **Find code duplication**:
   - Search for similar patterns
   - Look for repeated logic
   - Identify extraction opportunities

4. **Review constants usage**:
   - Check for magic numbers
   - Check for hardcoded strings
   - Verify constants are in application-constants.ts

5. **Identify SOLID violations**:
   - Single Responsibility: Classes doing too much
   - Open/Closed: Modification instead of extension
   - Liskov Substitution: Incorrect inheritance
   - Interface Segregation: Fat interfaces
   - Dependency Inversion: Tight coupling

### Report Format

```markdown
## Refactoring Opportunities

**Status**: ✅ CLEAN / ⚠️ NEEDS REFACTORING / ❌ CRITICAL

**Summary**:
- Large functions (>50 lines): [count]
- Deep nesting (>3 levels): [count]
- Code duplication: [X]%
- Magic numbers/strings: [count]
- Long parameter lists: [count]
- God objects: [count]

**Top Priorities**:

### 1. [Refactoring Need]
- **File**: `file.ts:line`
- **Issue**: [Description]
- **Impact**: HIGH/MEDIUM/LOW
- **Effort**: [Time estimate]
- **Recommendation**: [How to refactor]

### 2. [Next Priority]
[Same format]

**SOLID Principle Violations**:
1. [Violation description] - [File:line]
```

---

## Area 5: Testing Quality

### What to Check

#### Coverage Metrics

```bash
# Run tests with coverage
npm test -- --coverage

# Check coverage report
cat coverage/coverage-summary.json
```

**Coverage Targets**:
- **Statements**: 80%+
- **Branches**: 75%+
- **Functions**: 80%+
- **Lines**: 80%+

#### Test Quality Indicators

**Good Tests**:
```typescript
// ✅ Descriptive test names
test('should return empty array when product URL is not provided', () => {
  // Clear intent
})

// ✅ Arrange-Act-Assert pattern
test('should parse comma-separated values', () => {
  // Arrange
  const input = 'a,b,c'

  // Act
  const result = parse(input)

  // Assert
  expect(result).toEqual(['a', 'b', 'c'])
})

// ✅ Independent tests
beforeEach(() => {
  jest.resetAllMocks() // Clean slate
})

// ✅ Proper mocking
jest.spyOn(module, 'function').mockReturnValue(value)
```

**Bad Tests**:
```typescript
// ❌ Vague test names
test('test1', () => {})

// ❌ Testing multiple things
test('should work', () => {
  // Tests 5 different behaviors
})

// ❌ Shared state between tests
let sharedData
test('test1', () => {
  sharedData = 'value' // Affects other tests
})

// ❌ Over-mocking
jest.mock('everything')
// Tests implementation, not behavior
```

### Review Process

1. **Run coverage report**:
   ```bash
   npm test -- --coverage
   ```

2. **Analyze coverage by module**:
   - Check validators.ts: Should be 100%
   - Check utility.ts: Should be 90%+
   - Check bridge-cli.ts: Should be 80%+
   - Check services: Should be 80%+

3. **Review test structure**:
   - Are tests organized by module?
   - Do tests follow naming conventions?
   - Are there both unit and e2e tests?

4. **Check test quality**:
   - Descriptive test names
   - Arrange-Act-Assert pattern
   - Proper mocking strategy
   - Independent tests

5. **Identify missing tests**:
   - Uncovered branches
   - Edge cases
   - Error paths

### Report Format

```markdown
## Testing Quality

**Status**: ✅ EXCELLENT / ⚠️ NEEDS IMPROVEMENT / ❌ POOR

**Coverage Summary**:
- Overall: [X]%
- Statements: [X]%
- Branches: [X]%
- Functions: [X]%
- Lines: [X]%

**Coverage by Module**:
| Module | Coverage | Target | Status |
|--------|----------|--------|--------|
| validators.ts | [X]% | 100% | ✅/❌ |
| utility.ts | [X]% | 90% | ✅/❌ |
| bridge-cli.ts | [X]% | 80% | ✅/❌ |

**Missing Tests**:
1. [File:line] - [Uncovered code]
   - **Type**: Branch/Function/Statement
   - **Priority**: High/Medium/Low
   - **Recommendation**: [What tests to add]

**Test Quality Issues**:
1. [Test file] - [Issue description]

**Recommendations**:
- Add tests for uncovered branches
- Improve test naming
- Reduce test duplication
```

---

## Area 6: Design Patterns

### What to Check

#### Patterns in Use

Review these patterns in the codebase:

**1. Factory Pattern**:
- Location: `factory/github-client-service-factory.ts`
- Check: Is factory used appropriately?
- Validate: Runtime selection logic is correct

**2. Builder Pattern**:
- Location: `tools-parameter.ts`
- Check: Command building is clear
- Validate: Could benefit from fluent API?

**3. Strategy Pattern**:
- Location: `service/impl/`
- Check: Interface implementations are correct
- Validate: New strategies easy to add?

**4. Template Method Pattern**:
- Location: `service/impl/github-client-service-base.ts`
- Check: Base class provides good template
- Validate: Subclasses only customize what's needed

**5. Adapter Pattern**:
- Location: `artifacts.ts`
- Check: Adapts different API versions
- Validate: Handles both v1 and v2 correctly

#### Pattern Violations

**Missing Patterns**:
```typescript
// ❌ Could use Strategy Pattern
function validate(product: string) {
  if (product === 'polaris') {
    // Polaris validation
  } else if (product === 'coverity') {
    // Coverity validation
  }
  // Better with strategy pattern
}

// ✅ Strategy Pattern
interface ValidationStrategy {
  validate(): string[]
}

class PolarisValidation implements ValidationStrategy {
  validate(): string[] {
    // Polaris-specific validation
  }
}
```

**Anti-Patterns**:
1. **God Object**: Single class doing too much
2. **Lava Flow**: Dead code not removed
3. **Golden Hammer**: Overusing one pattern
4. **Spaghetti Code**: Tangled dependencies

### Review Process

1. **Identify patterns in use**:
   - Factory patterns
   - Builder patterns
   - Strategy patterns
   - Template method patterns
   - Adapter patterns

2. **Check pattern implementation**:
   - Is the pattern used correctly?
   - Does it solve the right problem?
   - Is it over-engineered?

3. **Find pattern opportunities**:
   - Where would patterns help?
   - What problems need abstraction?
   - Where is there duplication?

4. **Identify anti-patterns**:
   - God objects
   - Feature envy
   - Long parameter lists
   - Switch statements (could be strategy)

### Report Format

```markdown
## Design Patterns

**Status**: ✅ GOOD / ⚠️ VIOLATIONS FOUND / ❌ CRITICAL

**Patterns Implemented**:
1. **Factory Pattern** - `file.ts:line`
   - ✅ Used appropriately
   - ⚠️ Could be simplified

2. **Builder Pattern** - `file.ts:line`
   - ✅ Good structure
   - ⚠️ Opportunity for fluent API

**Pattern Violations**:
1. [Violation description]
   - **Location**: `file.ts:line`
   - **Issue**: [What's wrong]
   - **Recommendation**: [How to fix]

**Missing Patterns**:
1. **Strategy Pattern** for [use case]
   - **Location**: `file.ts:line`
   - **Benefit**: [Why it would help]

**Anti-Patterns Found**:
1. **God Object** in `tools-parameter.ts`
   - **Issue**: 990+ lines, too many responsibilities
   - **Recommendation**: Split into product-specific builders

**SOLID Principle Analysis**:
- Single Responsibility: [Assessment]
- Open/Closed: [Assessment]
- Liskov Substitution: [Assessment]
- Interface Segregation: [Assessment]
- Dependency Inversion: [Assessment]
```

---

## Area 7: Documentation

### What to Check

#### JSDoc Coverage

**Good Documentation**:
```typescript
/**
 * Validates Polaris input parameters
 *
 * Checks that all required parameters for Polaris scanning are provided
 * and valid. Only performs validation if Polaris is enabled (server URL set).
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
  // Implementation
}
```

**Missing Documentation**:
```typescript
// ❌ No JSDoc
export function validatePolarisInputs(): string[] {
  // What does this do?
}

// ❌ Incomplete JSDoc
/**
 * Validates inputs
 */
export function validatePolarisInputs(): string[] {
  // Too vague
}
```

#### Documentation Coverage Areas

1. **Public Functions**: 100% JSDoc coverage
2. **Public Classes**: 100% JSDoc coverage
3. **Public Methods**: 100% JSDoc coverage
4. **Interfaces**: 100% property descriptions
5. **Complex Logic**: Inline comments explaining why
6. **README Files**: Up-to-date and comprehensive

### Review Process

1. **Check JSDoc coverage**:
   ```bash
   # Find functions without JSDoc
   grep -B1 "export function" src/**/*.ts | grep -v "/**"
   ```

2. **Review documentation quality**:
   - Is description clear and helpful?
   - Are @param tags complete?
   - Are @returns tags descriptive?
   - Are @throws tags present for errors?
   - Are examples provided where helpful?

3. **Check inline comments**:
   - Complex logic explained?
   - Why, not what?
   - Up-to-date?

4. **Review README files**:
   - CLAUDE.md accurate?
   - README.md up-to-date?
   - Examples current?

### Report Format

```markdown
## Documentation

**Status**: ✅ WELL DOCUMENTED / ⚠️ GAPS FOUND / ❌ POOR

**Coverage Summary**:
- Public functions documented: [X]/[Total] ([X]%)
- Public classes documented: [X]/[Total] ([X]%)
- Public methods documented: [X]/[Total] ([X]%)
- Interfaces documented: [X]/[Total] ([X]%)

**Missing Documentation**:
1. [File:line] - [Function/Class name]
   - **Type**: Function/Class/Method
   - **Priority**: High/Medium/Low
   - **Recommendation**: [What to document]

**Documentation Quality Issues**:
1. [File:line] - [Issue description]
   - **Problem**: [What's wrong]
   - **Fix**: [How to improve]

**Inline Comment Issues**:
1. [File:line] - Outdated comment
2. [File:line] - Missing explanation for complex logic

**Recommendations**:
- Add JSDoc to all public functions
- Include @example for complex functions
- Update outdated comments
- Add inline comments for complex logic
```

---

## Comprehensive Review Process

### Step 1: Run Automated Checks

```bash
# Full quality pipeline
npm run all

# Individual checks
npm run format-check
npm run lint
npm test -- --coverage
npx tsc --noEmit
```

### Step 2: Manual Code Review

For each area:
1. Run automated tools
2. Manual inspection
3. Document findings
4. Prioritize issues

### Step 3: Generate Report

Combine all areas into comprehensive report:

```markdown
# Code Review Report

**Date**: [Date]
**Reviewer**: Code Review Skill
**Scope**: Full codebase review

## Executive Summary

### Overall Assessment
[High-level summary of code quality]

### Critical Issues: [Count]
### High Priority: [Count]
### Medium Priority: [Count]
### Low Priority: [Count]

## Detailed Findings

### 1. Linting & Formatting
[Full report from Area 1]

### 2. Type Safety
[Full report from Area 2]

### 3. Error Handling
[Full report from Area 3]

### 4. Refactoring Opportunities
[Full report from Area 4]

### 5. Testing Quality
[Full report from Area 5]

### 6. Design Patterns
[Full report from Area 6]

### 7. Documentation
[Full report from Area 7]

## Action Items

### Critical (Fix Immediately)
1. [Issue] - [File:line] - [Fix]

### High Priority (Fix This Sprint)
1. [Issue] - [File:line] - [Fix]

### Medium Priority (Fix Soon)
1. [Issue] - [File:line] - [Fix]

### Low Priority (Nice to Have)
1. [Issue] - [File:line] - [Fix]

## Recommendations

### Immediate Actions
- [Action 1]
- [Action 2]

### Short-term Actions
- [Action 1]
- [Action 2]

### Long-term Actions
- [Action 1]
- [Action 2]
```

---

## Best Practices

### Review Principles
- **Be objective**: Focus on code, not author
- **Be specific**: Provide file:line references
- **Be constructive**: Suggest solutions, not just problems
- **Be comprehensive**: Cover all seven areas
- **Prioritize**: Critical > High > Medium > Low

### When to Run
- Before major releases
- After significant changes
- Weekly as part of development
- Before code merges

### Tools to Use
- ESLint for linting
- Prettier for formatting
- TypeScript compiler for type checking
- Jest for test coverage
- Manual inspection for patterns and design

---

## Example Usage

**User**: "Review the code quality"

**Response**:
1. Run automated checks (npm run all)
2. Review each of 7 areas
3. Generate comprehensive report
4. Prioritize action items
5. Provide recommendations
