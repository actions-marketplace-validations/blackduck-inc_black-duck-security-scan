---
name: code-generation
description: Comprehensive code generation skill that generates new TypeScript components, modules, services, utilities, tests, and documentation following Black Duck Security Scan codebase patterns and best practices. Ensures type safety, proper error handling, consistent code style, and comprehensive test coverage.
---

# Code Generation

Generates production-ready TypeScript code for the Black Duck Security Scan GitHub Action, including components, tests, documentation, and type-safe implementations.

## Usage

Run this skill when the user requests:
- "Generate a new [component/service/validator/utility]"
- "Create tests for [feature]"
- "Add documentation for [module]"
- "Make this code type-safe"
- "Generate a validator for [product]"
- "Create a new service for [feature]"
- "Write tests to cover [functionality]"

## Comprehensive Code Generation

This skill performs complete code generation covering:
1. Component Generation (models, validators, services, utilities)
2. Test Generation (unit and E2E tests)
3. Documentation Generation (JSDoc, README, inline comments)
4. Type-Safe Code (strict TypeScript compliance)

---

## Section 1: Component Generation

### Code Generation Principles

#### 1. Type Safety (Strict TypeScript)
- **Explicit types**: Always define return types and parameter types
- **Interfaces over types**: Use interfaces for object shapes
- **No implicit any**: Avoid implicit `any` types
- **Null safety**: Handle null/undefined explicitly with `?` or `||`
- **Type guards**: Create runtime type checking for validation

#### 2. Error Handling
- **Try-catch blocks**: Wrap operations that can fail
- **Error transformation**: Convert generic errors to descriptive errors
- **Error propagation**: Re-throw with context when appropriate
- **Validation errors**: Collect errors in arrays, don't fail fast

#### 3. Code Structure
- **Single responsibility**: One module, one purpose
- **Separation of concerns**: Keep validation, transformation, execution separate
- **Constants**: Extract magic strings/numbers to application-constants.ts
- **Exports**: Export only what's needed

#### 4. Formatting & Style
- **Prettier configuration**:
  - No semicolons
  - Single quotes
  - 2-space indentation
  - No trailing commas
  - Arrow functions: omit parens for single params
- **Naming conventions**:
  - camelCase for variables and functions
  - PascalCase for classes and interfaces
  - UPPER_CASE for constants (in application-constants.ts)

### Component Templates

#### Template 1: Input Data Model

**Use Case**: Adding a new security product or extending existing product configuration

**Location**: `src/blackduck-security-action/input-data/`

```typescript
import {Common} from './common'
import {Network} from './common'
import {Bridge} from './bridge'
import {GithubData} from './github'

/**
 * [Product Name] input data structure for Bridge CLI
 */
export interface [ProductName] {
  [productKey]: [ProductName]Data
  network: Network
  bridge: Bridge
  github?: GithubData
}

/**
 * [Product Name] specific configuration
 */
export interface [ProductName]Data extends Common {
  /**
   * Access token for authentication
   */
  accesstoken: string

  /**
   * Server URL for [Product Name]
   */
  serverUrl: string

  /**
   * [Product-specific field description]
   */
  [field]: {[subField]: string}
}

/**
 * [Sub-configuration description]
 */
export interface [SubConfiguration] {
  [property]: string | boolean | string[]
}
```

**Checklist**:
- ✅ Extends Common interface for shared fields
- ✅ Includes Network and Bridge for standard config
- ✅ Optional GitHub for integration features
- ✅ JSDoc comments for all interfaces and properties
- ✅ Descriptive property names matching Bridge CLI expectations
- ✅ Union types for enums (e.g., 'DAST' | 'IAST')

---

#### Template 2: Validator Function

**Use Case**: Adding validation for new product or extending existing validation

**Location**: `src/blackduck-security-action/validators.ts`

```typescript
/**
 * Validates [Product Name] input parameters
 *
 * @returns Array of error messages (empty if valid)
 */
export function validate[ProductName]Inputs(): string[] {
  let errors: string[] = []

  // Only validate if product is enabled
  if (inputs.[PRODUCT_URL_KEY]) {
    const paramsMap = new Map<string, string>()

    // Add required parameters
    paramsMap.set(constants.[PARAM1_KEY], inputs.[PARAM1])
    paramsMap.set(constants.[PARAM2_KEY], inputs.[PARAM2])

    // Validate parameters
    errors = validateParameters(paramsMap, constants.[PRODUCT_KEY])

    // Add custom validation logic if needed
    if (inputs.[CUSTOM_PARAM]) {
      if (![CONDITION]) {
        errors.push(`[${constants.[CUSTOM_PARAM_KEY]}] is invalid: [reason]`)
      }
    }
  }

  return errors
}
```

**Checklist**:
- ✅ Return type: `string[]`
- ✅ Conditional validation (only if product enabled)
- ✅ Use validateParameters() for standard checks
- ✅ Custom validation for product-specific rules
- ✅ Error messages reference constant keys
- ✅ JSDoc comment explaining purpose

---

#### Template 3: Service Implementation

**Use Case**: Adding new service or extending GitHub client services

**Location**: `src/blackduck-security-action/service/impl/`

```typescript
import {[BaseService]} from './[base-service]'
import {I[Service]} from '../[service-interface]'

/**
 * [Service description]
 *
 * Handles [specific responsibility]
 */
export class [ServiceName] extends [BaseService] implements I[Service] {
  /**
   * [Private property description]
   */
  private readonly config: [ConfigType]

  /**
   * Constructor
   *
   * @param config [Configuration description]
   */
  constructor(config: [ConfigType]) {
    super()
    this.config = config
  }

  /**
   * [Method description]
   *
   * @param param [Parameter description]
   * @returns [Return value description]
   * @throws {Error} If [error condition]
   */
  async [methodName](param: [ParamType]): Promise<[ReturnType]> {
    try {
      // Implementation

      // Use parent class methods if available
      await this.[baseMethod]([args])

      return [result]
    } catch (error) {
      // Transform error with context
      throw new Error(`Failed to [action]: ${error}`)
    }
  }
}
```

**Checklist**:
- ✅ Extends appropriate base class
- ✅ Implements interface
- ✅ JSDoc for class and all public methods
- ✅ Async/await for async operations
- ✅ Try-catch with descriptive errors
- ✅ Use base class methods when available
- ✅ Explicit return types

---

#### Template 4: Utility Function

**Use Case**: Adding reusable helper functions

**Location**: `src/blackduck-security-action/utility.ts` or new utility file

```typescript
/**
 * [Function description]
 *
 * @param param1 [Description]
 * @param param2 [Description]
 * @returns [Description]
 * @throws {TypeError} [When it throws]
 */
export function [functionName](
  param1: [Type1],
  param2: [Type2]
): [ReturnType] {
  // Input validation
  if (![validation]) {
    throw new TypeError(`[param] is required for [operation]`)
  }

  try {
    // Implementation

    return [result]
  } catch (error) {
    // Handle error appropriately
    throw new Error(`Failed to [operation]: ${error}`)
  }
}
```

**Checklist**:
- ✅ JSDoc with @param, @returns, @throws
- ✅ Input validation
- ✅ Explicit return type
- ✅ Error handling (try-catch or validation)
- ✅ Meaningful variable names
- ✅ Single responsibility

---

#### Template 5: Command Builder Method

**Use Case**: Adding command building for new product in tools-parameter.ts

**Location**: `src/blackduck-security-action/tools-parameter.ts`

```typescript
/**
 * Builds Bridge CLI command for [Product Name]
 *
 * @returns Array of CLI arguments
 */
public getFormattedCommandFor[ProductName](): string[] {
  const [productKey]Data: InputData<[ProductName]> = {
    data: {
      [productKey]: this.get[ProductName]Data(),
      network: this.getNetworkData(),
      bridge: this.getBridgeData()
    }
  }

  // Add optional sections
  if ([condition]) {
    [productKey]Data.data.github = this.getGithubRepoInfo()
  }

  // Write JSON input file
  const inputFileName = `${constants.[PRODUCT_INPUT_FILE]}`
  const inputFilePath = path.join(this.tempDir, inputFileName)
  fs.writeFileSync(inputFilePath, JSON.stringify([productKey]Data))

  // Build command arguments
  const formattedCommand: string[] = []
  formattedCommand.push(constants.SPACE)
  formattedCommand.push(constants.STAGE_OPTION)
  formattedCommand.push(constants.[PRODUCT_STAGE_NAME])
  formattedCommand.push(constants.SPACE)
  formattedCommand.push(constants.INPUT_OPTION)
  formattedCommand.push(inputFilePath)

  return formattedCommand
}

/**
 * Builds [Product Name] data object from inputs
 *
 * @returns [ProductName]Data object
 */
private get[ProductName]Data(): [ProductName]Data {
  const [productKey]Data: [ProductName]Data = {
    accesstoken: inputs.[PRODUCT_ACCESS_TOKEN],
    serverUrl: inputs.[PRODUCT_SERVER_URL]
    // Add required fields
  }

  // Add optional fields
  if (inputs.[OPTIONAL_FIELD]) {
    [productKey]Data.[optionalField] = {
      [property]: inputs.[OPTIONAL_FIELD]
    }
  }

  return [productKey]Data
}
```

---

### Type Safety Patterns

#### Explicit Types
```typescript
// ✅ GOOD: Explicit types everywhere
function process(input: string): string[] {
  const results: string[] = []
  // ...
  return results
}

// ❌ BAD: Implicit types
function process(input) {
  const results = []
  return results
}
```

#### Null Safety
```typescript
// ✅ GOOD: Null handling
const value = input?.trim() ?? ''
if (value.length === 0) {
  return []
}

// ❌ BAD: No null handling
const value = input.trim()
```

#### Type Guards
```typescript
/**
 * Type guard for custom type
 */
function isError(value: unknown): value is Error {
  return value instanceof Error
}

function processError(error: unknown): string {
  if (isError(error)) {
    return error.message
  }
  return 'Unknown error'
}
```

---

## Section 2: Test Generation

### Test Types

**Unit Tests** (`test/unit/**/*.test.ts`)
- Test individual functions/classes in isolation
- Heavy use of mocks to isolate units
- Coverage: Business logic, validation, transformations

**Contract/E2E Tests** (`test/contract/**/*.e2e.test.ts`)
- Test integration scenarios end-to-end
- Minimal mocking, test actual workflows
- Coverage: Product workflows, Bridge CLI execution

### Test Templates

#### Template 1: Unit Test for Validator

```typescript
import * as validators from '../../../src/blackduck-security-action/validators'
import * as inputs from '../../../src/blackduck-security-action/inputs'
import * as constants from '../../../src/application-constants'

jest.mock('../../../src/blackduck-security-action/inputs')

describe('validate[ProductName]Inputs', () => {
  beforeEach(() => {
    jest.resetAllMocks()
  })

  afterEach(() => {
    jest.restoreAllMocks()
  })

  test('should return empty array when all required inputs are provided', () => {
    // Arrange
    Object.defineProperty(inputs, '[PRODUCT_URL_KEY]', {
      value: 'https://example.com',
      writable: true
    })
    Object.defineProperty(inputs, '[PRODUCT_TOKEN_KEY]', {
      value: 'test-token',
      writable: true
    })

    // Act
    const errors = validators.validate[ProductName]Inputs()

    // Assert
    expect(errors).toEqual([])
    expect(errors.length).toBe(0)
  })

  test('should return error array when required input is missing', () => {
    // Arrange
    Object.defineProperty(inputs, '[PRODUCT_URL_KEY]', {
      value: 'https://example.com',
      writable: true
    })
    Object.defineProperty(inputs, '[PRODUCT_TOKEN_KEY]', {
      value: '',
      writable: true
    })

    // Act
    const errors = validators.validate[ProductName]Inputs()

    // Assert
    expect(errors.length).toBeGreaterThan(0)
    expect(errors).toContain(
      expect.stringContaining(constants.[PRODUCT_TOKEN_KEY])
    )
  })

  test('should return empty array when product is not enabled', () => {
    // Arrange
    Object.defineProperty(inputs, '[PRODUCT_URL_KEY]', {
      value: '',
      writable: true
    })

    // Act
    const errors = validators.validate[ProductName]Inputs()

    // Assert
    expect(errors).toEqual([])
  })
})
```

#### Template 2: Unit Test for Utility

```typescript
import * as utility from '../../../src/blackduck-security-action/utility'

describe('[functionName]', () => {
  test('should [expected behavior] when [condition]', () => {
    // Arrange
    const input = [test data]

    // Act
    const result = utility.[functionName](input)

    // Assert
    expect(result).toBe([expected])
  })

  test('should handle null/undefined input', () => {
    // Arrange
    const input = null

    // Act & Assert
    expect(() => utility.[functionName](input)).toThrow([expected error])
  })

  test('should handle empty input', () => {
    const input = ''
    const result = utility.[functionName](input)
    expect(result).toEqual([])
  })
})
```

#### Template 3: Contract/E2E Test

```typescript
import {run} from '../../src/main'
import * as inputs from '../../src/blackduck-security-action/inputs'

jest.mock('@actions/core')

describe('[Product] E2E Tests', () => {
  beforeEach(() => {
    jest.resetAllMocks()
  })

  afterEach(() => {
    jest.restoreAllMocks()
  })

  test('should execute successfully with all mandatory fields', async () => {
    // Arrange: Setup all required inputs
    Object.defineProperty(inputs, '[PRODUCT]_SERVER_URL', {
      value: 'https://example.com',
      writable: true
    })

    // Mock Bridge CLI execution
    jest.spyOn(require('@actions/exec'), 'exec').mockResolvedValue(0)

    // Act
    const result = await run()

    // Assert
    expect(result).toBeUndefined()
  })

  test('should fail with missing mandatory field', async () => {
    // Arrange: Missing required field
    Object.defineProperty(inputs, '[PRODUCT]_SERVER_URL', {
      value: 'https://example.com',
      writable: true
    })

    // Act & Assert
    await expect(run()).rejects.toThrow()
  })
})
```

### Test Best Practices

#### Arrange-Act-Assert Pattern
```typescript
test('should parse comma-separated values', () => {
  // Arrange: Set up test data
  const input = 'value1,value2,value3'

  // Act: Execute the code under test
  const result = parseCommaSeparated(input)

  // Assert: Verify the result
  expect(result).toEqual(['value1', 'value2', 'value3'])
})
```

#### Mocking
```typescript
// Mock entire module
jest.mock('../../../src/blackduck-security-action/inputs')

// Mock specific function
jest.spyOn(module, 'functionName').mockReturnValue(mockValue)

// Mock async function
jest.spyOn(module, 'asyncFunction').mockResolvedValue(mockValue)

// Mock property
Object.defineProperty(inputs, 'PARAM', {
  value: 'test-value',
  writable: true
})
```

---

## Section 3: Documentation Generation

### JSDoc Standards

#### Function Documentation
```typescript
/**
 * [Brief one-line description]
 *
 * [Detailed description if needed, explaining what the function does,
 * when to use it, and any important notes]
 *
 * @param param1 [Description of first parameter]
 * @param param2 [Description of second parameter]
 * @returns [Description of return value]
 * @throws {ErrorType} [When this error is thrown]
 *
 * @example
 * ```typescript
 * const result = functionName('input', 42)
 * console.log(result) // ['output1', 'output2']
 * ```
 */
export function functionName(
  param1: string,
  param2: number
): string[] {
  // Implementation
}
```

#### Interface Documentation
```typescript
/**
 * [Brief description of what this interface represents]
 *
 * [Additional details about usage, relationships to other types, etc.]
 */
export interface DataModel {
  /**
   * [Property description]
   */
  requiredField: string

  /**
   * [Optional property description]
   *
   * @default undefined
   */
  optionalField?: number

  /**
   * [Nested object description]
   */
  config: {
    /**
     * [Sub-property description]
     */
    enabled: boolean
  }
}
```

#### Class Documentation
```typescript
/**
 * [Class description - what it does and when to use it]
 *
 * [Additional details about responsibilities, usage patterns, etc.]
 *
 * @example
 * ```typescript
 * const service = new ServiceName(config)
 * await service.execute()
 * ```
 */
export class ServiceName {
  /**
   * [Private property description]
   */
  private readonly config: ConfigType

  /**
   * Creates a new instance of [ServiceName]
   *
   * @param config [Configuration description]
   */
  constructor(config: ConfigType) {
    this.config = config
  }

  /**
   * [Method description]
   *
   * @param param [Parameter description]
   * @returns [Return value description]
   * @throws {Error} If [error condition]
   */
  async methodName(param: ParamType): Promise<ReturnType> {
    // Implementation
  }
}
```

---

## Section 4: Type-Safe Refactoring

### Making Code Type-Safe

**Before** (not type-safe):
```typescript
function process(data) {
  const value = data.value || ''
  return value.split(',')
}
```

**After** (type-safe):
```typescript
interface InputData {
  value: string | null | undefined
}

/**
 * Processes input data by splitting comma-separated values
 *
 * @param data Input data containing comma-separated values
 * @returns Array of trimmed values
 */
function process(data: InputData): string[] {
  // Null safety with type narrowing
  const value = data.value?.trim() ?? ''

  // Early return for empty input
  if (value.length === 0) {
    return []
  }

  // Type-safe operations
  return value.split(',').map(s => s.trim())
}
```

---

## Generation Workflow

### Step 1: Understand Requirements
- What is the component's purpose?
- What inputs does it need?
- What outputs does it produce?
- What errors can it throw?

### Step 2: Choose Template
- Data model → Template 1
- Validation → Template 2
- Service → Template 3
- Utility → Template 4
- Command building → Template 5

### Step 3: Generate Code
- Follow appropriate template
- Use descriptive names
- Add JSDoc comments
- Include error handling
- Apply proper types

### Step 4: Generate Tests
- Create unit tests for logic
- Create contract tests for integration
- Follow Arrange-Act-Assert pattern
- Mock dependencies appropriately

### Step 5: Add Documentation
- JSDoc for all public APIs
- Inline comments for complex logic
- Examples where helpful

### Step 6: Register Component
- Add exports to index files if needed
- Update validators if adding new product
- Update bridge-cli.ts if adding new product
- Update action.yml if adding new inputs
- Add constants to application-constants.ts

---

## Code Generation Checklist

### Type Safety
- ✅ All parameters have explicit types
- ✅ All return types are explicit
- ✅ No `any` types (use `unknown` if truly unknown)
- ✅ Interfaces for object shapes
- ✅ Null/undefined handled explicitly

### Error Handling
- ✅ Try-catch for operations that can fail
- ✅ Error transformation with context
- ✅ Validation errors collected in arrays
- ✅ Descriptive error messages

### Code Style
- ✅ No semicolons
- ✅ Single quotes
- ✅ 2-space indentation
- ✅ camelCase for variables/functions
- ✅ PascalCase for classes/interfaces

### Documentation
- ✅ JSDoc for all public APIs
- ✅ @param, @returns, @throws tags
- ✅ Examples where helpful
- ✅ Inline comments for complex logic

### Testing
- ✅ Unit tests for all logic
- ✅ Contract tests for integration
- ✅ Arrange-Act-Assert pattern
- ✅ Proper mocking

---

## Best Practices Summary

### DO:
- ✅ Use strict TypeScript types
- ✅ Add JSDoc comments for all public APIs
- ✅ Handle errors explicitly
- ✅ Extract constants to application-constants.ts
- ✅ Follow existing naming conventions
- ✅ Keep functions focused and small (<50 lines)
- ✅ Validate inputs early
- ✅ Use interfaces for object shapes
- ✅ Write tests for all new code

### DON'T:
- ❌ Use `any` type
- ❌ Hard-code strings or magic numbers
- ❌ Create large functions (>100 lines)
- ❌ Mix concerns (validation + execution + transformation)
- ❌ Swallow errors without logging
- ❌ Skip documentation
- ❌ Skip tests
- ❌ Use semicolons, double quotes, or tabs
