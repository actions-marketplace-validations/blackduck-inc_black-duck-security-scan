---
name: task-analyzer
description: Analyzes implementation requirements for new features in the Black Duck Security Scan codebase. Reads the codebase, maps every file that must change, identifies breakage risks, dependencies, and writes a structured implementation plan with prioritized changes and risk assessment. Use when planning feature implementations or analyzing impact of proposed changes.
---

# Task Analyzer

Provides comprehensive analysis of what changes are needed to implement new features or modifications in the Black Duck Security Scan GitHub Action codebase.

## Usage

Run this skill when the user asks:
- "Analyze what needs to change for [feature]"
- "What files need to change to implement [feature]?"
- "Plan the implementation for [feature]"
- "Impact analysis for [feature]"
- "What changes are needed for [feature]?"
- "Analyze this feature implementation"
- "Create implementation plan for [feature]"

## How to Use

When analyzing a feature request:

1. **Parse the request** — Extract what, scope, trigger, output
2. **Read current state** — Review relevant source files first
3. **Map full change surface** — Identify ALL files that must change
4. **Identify breakage risks** — Use Risk ID system (R1-R10)
5. **Write structured output** — Generate implementation plan with risk assessment

**Never guess** — read source files before mapping changes.

---

## Step 1 — Parse the Feature Request

Extract from the user's request:

- **What** — the capability being added (new product, new param, behavior change, etc.)
- **Scope** — which product(s) affected (Polaris, Coverity, Black Duck SCA, SRM, all)
- **Trigger** — GitHub Action input, Bridge CLI version, environment variable
- **Output** — what changes at runtime (new JSON field to Bridge CLI, new artifact, new API call)

If any of these is unclear, **ask before proceeding**.

**Example**:
```
Feature: Add support for a new security product "Seeker"
- What: New DAST/IAST product integration
- Scope: New product (similar to Polaris, Coverity)
- Trigger: GitHub Action inputs (seeker_server_url, seeker_api_token)
- Output: JSON input file for Bridge CLI, SARIF report
```

---

## Step 2 — Read Current State

Before mapping changes, **read the relevant source files**. Use this dependency map:

```
Feature type                → Read these files first
─────────────────────────────────────────────────────────────────────
New Action input            → inputs.ts, application-constants.ts, action.yml
New product                 → input-data/{closest product}.ts, tools-parameter.ts, validators.ts
Behavior change in scan     → tools-parameter.ts → getFormattedCommandFor{Product}()
Bridge CLI version gate     → utility.ts → version helpers, application-constants.ts → VERSION constants
SARIF / report change       → main.ts (SARIF upload), utility.ts → updatePolarisSarifPath()
GitHub API change           → service/impl/, factory/
Proxy / SSL change          → proxy-utils.ts, ssl-utils.ts
Dependency upgrade          → package.json, package-lock.json, dist/
Error handling              → application-constants.ts → EXIT_CODE_MAP, main.ts error handling
```

---

## Step 3 — Map the Full Change Surface

For each changed file, determine:
- **Why** it must change (not just "it's related")
- **What specifically** changes (new export, new field, new condition)
- **Risk level** — see Risk IDs below

Use this canonical file inventory to check every layer:

### Layer 1 — Constants and Types (always check first)

| File | Change if |
|---|---|
| `application-constants.ts` | New input key, exit code, version threshold, or message string needed |
| `input-data/{product}.ts` | New field in product config sent to Bridge CLI |

### Layer 2 — Input and Validation

| File | Change if |
|---|---|
| `inputs.ts` | New input parameter exported |
| `validators.ts` | New required-field check, new allowed-value list, cross-param validation |

### Layer 3 — Command Building

| File | Change if |
|---|---|
| `tools-parameter.ts` | New field in JSON input file, new `--stage` flag, new product command builder |
| `bridge-cli.ts` | New product stage wired into `prepareCommand()`, new version detection |

### Layer 4 — Orchestration and Output

| File | Change if |
|---|---|
| `main.ts` | New SARIF upload block, new artifact logic, new build status path |
| `artifacts.ts` | New artifact type uploaded |
| `service/` | New GitHub API call, new service implementation |
| `factory/` | New service selection logic |
| `utility.ts` | New version helper, new path resolution, new HTTP client variant |

### Layer 5 — Action Metadata

| File | Change if |
|---|---|
| `action.yml` | New input, version bump |
| `package.json` | Dependency change, version bump |
| `README.md` | New feature documentation, usage examples |

### Layer 6 — Tests

| File | Change if |
|---|---|
| `test/unit/**/*.test.ts` | Any logic change in corresponding module |
| `test/contract/**/*.e2e.test.ts` | New product workflow, integration scenario |

---

### Core Changes (Required)

**1. Action Definition** (`action.yml`)
- Add new inputs for the product
- Document input descriptions
- Set required/optional flags

**2. Constants** (`src/application-constants.ts`)
- Add product name constants
- Add parameter key constants
- Add stage name for Bridge CLI
- Add input file name

**3. Inputs** (`src/blackduck-security-action/inputs.ts`)
- Read new action inputs
- Handle deprecated parameter names if applicable

**4. Input Data Model** (`src/blackduck-security-action/input-data/[product].ts`)
- Create new interface for product data
- Define product-specific configuration
- Extend Common interface

**5. Validators** (`src/blackduck-security-action/validators.ts`)
- Create validation function for new product
- Validate required parameters
- Add custom validation rules

**6. Command Builder** (`src/blackduck-security-action/tools-parameter.ts`)
- Add command building method
- Create JSON input file generator
- Build CLI arguments

**7. Bridge CLI Orchestrator** (`src/blackduck-security-action/bridge-cli.ts`)
- Add product to validation check
- Include product in command preparation
- Update scan type detection

#### Supporting Changes (If Needed)

**8. GitHub Integration** (if SARIF/PR comments needed)
- Update SARIF report handling
- Add PR comment support
- Update artifact upload logic

**9. Utilities** (if new helpers needed)
- Add product-specific utility functions
- Path handling for SARIF reports
- Version compatibility checks

**10. Documentation**
- Update README.md
- Update CLAUDE.md
- Add usage examples

**11. Tests**
- Unit tests for validators
- Unit tests for command builder
- Contract/E2E tests for full workflow
- Mock data for testing

---

## Step 4 — Identify Breakage Risks

For each change, assess against these risk categories:

| Risk ID | Risk | Triggered by | Severity |
|---|---|---|---|
| R1 | **Workflow breakage** | Removing/renaming input without deprecation, making optional param required | HIGH |
| R2 | **Bridge CLI version breakage** | New behavior without version gate, assumed universal support | HIGH |
| R3 | **Build status breakage** | New exit code not in EXIT_CODE_MAP, changed mark_build_status logic | MEDIUM |
| R4 | **SARIF upload failure** | Path not version-gated (v1 vs v2.0+), incorrect path resolution | MEDIUM |
| R5 | **Air-gap mode breakage** | New download operation not gated behind `!NETWORK_AIRGAP` check | HIGH |
| R6 | **Credential exposure** | Secret logged, written to workspace, not masked with core.setSecret() | CRITICAL |
| R7 | **GitHub API breakage** | Incorrect service selection, Cloud vs Enterprise compatibility | MEDIUM |
| R8 | **Test regression** | Model interface changed, inputs renamed, validator return type changed | LOW |
| R9 | **Action metadata mismatch** | action.yml inputs don't match application-constants.ts keys | HIGH |
| R10 | **Dependency breakage** | New dependency not in dist/, incompatible version, missing in package-lock.json | MEDIUM |

---

### Step 5: Identify Dependencies

**Internal Dependencies**:
- What existing code does this depend on?
- What patterns should be followed?
- What interfaces need to be implemented?

**External Dependencies**:
- Bridge CLI version requirements
- New npm packages needed?
- GitHub API changes?

**Pattern Dependencies**:
- Factory pattern (for services)
- Builder pattern (for commands)
- Strategy pattern (for product-specific logic)

---

### Step 4: Risk Assessment

Identify potential risks and breaking changes:

#### High Risk
- Changes to existing product behavior
- Modifications to core execution flow
- Breaking changes to action inputs
- Changes to output format

#### Medium Risk
- New validation logic affecting existing products
- Changes to error handling
- New dependencies with compatibility issues

#### Low Risk
- Adding new product (following existing patterns)
- Adding optional inputs
- Documentation updates
- Test additions

---

### Step 5: Implementation Plan

Create step-by-step implementation plan:

#### Phase 1: Foundation (Low Risk)
1. Add constants
2. Create input data model
3. Add inputs
4. Write tests for validation

#### Phase 2: Core Logic (Medium Risk)
5. Implement validator
6. Build command builder
7. Update Bridge CLI orchestrator
8. Write unit tests

#### Phase 3: Integration (Medium Risk)
9. Add to action.yml
10. Update GitHub integration (if needed)
11. Add utilities (if needed)
12. Write E2E tests

#### Phase 4: Documentation (Low Risk)
13. Update README
14. Update CLAUDE.md
15. Add usage examples

---

## Step 6 — Generate Implementation Plan

Output a structured implementation plan document:

```markdown
# Implementation Plan: [Feature Name]

## Summary
[One paragraph: what the feature does, which products it affects, what changes at runtime]

## Feature Scope
- **Type**: [New product | New parameter | Behavior change | Version gate | Integration]
- **Products affected**: [Polaris | Coverity | Black Duck SCA | SRM | All | N/A]
- **Trigger**: [Action input | Bridge CLI version | Environment variable]
- **Runtime output**: [New JSON field to Bridge CLI | New artifact | New API call]

---

## Files to Change

### Must Change
| File | Change | Risk |
|---|---|---|
| `src/application-constants.ts` | Add `MYPRODUCT_SERVER_URL_KEY`, `MYPRODUCT_ACCESS_TOKEN_KEY` constants | R9 |
| `src/blackduck-security-action/inputs.ts` | Export `MYPRODUCT_SERVER_URL`, `MYPRODUCT_ACCESS_TOKEN` | R1 |
| `src/blackduck-security-action/input-data/myproduct.ts` | Create `MyProductData` interface extending `Common` | R8 |
| `src/blackduck-security-action/validators.ts` | Add `validateMyProductInputs(): string[]` | R1, R8 |
| `src/blackduck-security-action/tools-parameter.ts` | Add `getFormattedCommandForMyProduct()`, write `myproduct_input.json` | R2, R4 |
| `src/blackduck-security-action/bridge-cli.ts` | Call `validateMyProductInputs()` in `prepareCommand()` | R1 |
| `action.yml` | Add `myproduct_server_url`, `myproduct_access_token` inputs | R9 |

### May Change (conditional on final design)
| File | Change | Condition |
|---|---|---|
| `src/main.ts` | Add SARIF upload for MyProduct | Only if product generates SARIF |
| `src/blackduck-security-action/utility.ts` | Add MyProduct-specific SARIF path resolution | Only if SARIF path differs from standard |

---

## Breakage Risk Assessment

| Risk ID | Risk | Affected scenario | Mitigation |
|---|---|---|---|
| R1 | Workflow breakage | N/A (new product, no existing users) | None needed |
| R2 | Bridge CLI version | Users on Bridge CLI < X.Y.Z won't have MyProduct support | Document minimum Bridge CLI version, add version check if possible |
| R4 | SARIF upload failure | MyProduct SARIF path might differ | Test with both Bridge CLI v1 and v2.0+, verify path detection |
| R8 | Test regression | New interfaces could break tests | Add comprehensive test coverage before merge |
| R9 | Action metadata mismatch | action.yml keys must match constants | Double-check naming consistency |

**Overall risk**: [LOW | MEDIUM | HIGH]
**Reason**: [One sentence on the dominant risk factor]

---

## Implementation Order

Steps in the correct dependency order (later steps depend on earlier):

1. `application-constants.ts` — add constants (all other files import from here)
2. `input-data/myproduct.ts` — create data model interface
3. `inputs.ts` — export new input constants
4. `validators.ts` — add validation function
5. `tools-parameter.ts` — add command builder method
6. `bridge-cli.ts` — wire into prepareCommand()
7. `action.yml` — add action inputs
8. `test/unit/` — add unit tests for validator and command builder
9. `test/contract/` — add E2E test for full workflow
10. `npm run all` — verify build, lint, tests pass

---

## Open Questions

[List any decisions that must be made before implementation starts. If none, write "None."]

- [ ] What is the minimum Bridge CLI version that supports MyProduct?
- [ ] Does MyProduct generate SARIF reports? If yes, what's the output path?
- [ ] Are there any MyProduct-specific configuration fields beyond URL/token?

---

### Files to Create (New)
1. **`src/blackduck-security-action/input-data/[product].ts`**
   - Purpose: Define product data model
   - Complexity: LOW
   - Lines: ~50-100
   - Dependencies: common.ts, bridge.ts

2. **`test/unit/blackduck-security-action/validators/[product].test.ts`**
   - Purpose: Unit tests for validator
   - Complexity: LOW
   - Lines: ~100-200
   - Dependencies: validators.ts

3. **`test/contract/[product].e2e.test.ts`**
   - Purpose: E2E tests for full workflow
   - Complexity: MEDIUM
   - Lines: ~100-150
   - Dependencies: main.ts

### Files to Modify (Existing)

1. **`action.yml`** ⚠️ HIGH IMPACT
   - Changes:
     - Add [X] new inputs
     - Document each input
   - Lines affected: +[X] lines
   - Risk: LOW (backward compatible)
   - Breaking: NO

2. **`src/application-constants.ts`** ✅ LOW RISK
   - Changes:
     - Add [X] constant definitions
     - Add product key
     - Add stage name
   - Lines affected: +[X] lines
   - Risk: LOW
   - Breaking: NO

3. **`src/blackduck-security-action/inputs.ts`** ✅ LOW RISK
   - Changes:
     - Read new inputs
     - Export constants
   - Lines affected: +[X] lines
   - Risk: LOW
   - Breaking: NO

4. **`src/blackduck-security-action/validators.ts`** ⚠️ MEDIUM RISK
   - Changes:
     - Add validation function
     - Implement custom validation rules
   - Lines affected: +[X] lines
   - Risk: MEDIUM (validation logic)
   - Breaking: NO

5. **`src/blackduck-security-action/tools-parameter.ts`** ⚠️ HIGH RISK
   - Changes:
     - Add command builder method
     - Generate JSON input file
     - Build CLI arguments
   - Lines affected: +[X] lines
   - Risk: MEDIUM (complex file)
   - Breaking: NO

6. **`src/blackduck-security-action/bridge-cli.ts`** ⚠️ HIGH RISK
   - Changes:
     - Add product to validation check
     - Update scan type detection
   - Lines affected: ~[X] lines modified
   - Risk: MEDIUM (core execution)
   - Breaking: NO

7. **`README.md`** ✅ LOW RISK
   - Changes:
     - Add usage documentation
     - Add examples
   - Lines affected: +[X] lines
   - Risk: LOW
   - Breaking: NO

8. **`CLAUDE.md`** ✅ LOW RISK
   - Changes:
     - Document new product
     - Update architecture notes
   - Lines affected: +[X] lines
   - Risk: LOW
   - Breaking: NO

---

## Dependencies

### Internal Dependencies
- **Common Interface**: Extends existing Common interface
- **Network Interface**: Uses existing Network configuration
- **Bridge Interface**: Uses existing Bridge configuration
- **Validation Pattern**: Follows validateParameters() pattern
- **Command Builder Pattern**: Follows existing product builders

### External Dependencies
- **Bridge CLI**: Version [X.Y.Z]+ required
- **@actions/core**: Existing
- **@actions/exec**: Existing
- **No new npm packages required**: ✅

### Pattern Requirements
- Factory Pattern: Not needed (no service creation)
- Builder Pattern: Existing tools-parameter.ts pattern
- Strategy Pattern: Not needed (follows existing product pattern)

---

## Risk Assessment

### High Risk Changes
None expected - Following established patterns

### Medium Risk Changes

1. **Validator Logic** (`validators.ts`)
   - Risk: New validation might affect existing validation flow
   - Mitigation: Keep validation isolated, return error arrays
   - Testing: Extensive unit tests

2. **Command Builder** (`tools-parameter.ts`)
   - Risk: Large file (990+ lines), complex logic
   - Mitigation: Follow existing pattern exactly
   - Testing: Unit tests + E2E tests

3. **Bridge CLI Integration** (`bridge-cli.ts`)
   - Risk: Core execution flow modification
   - Mitigation: Minimal changes, add to existing checks
   - Testing: Contract tests for all products

### Low Risk Changes

1. **Constants** (`application-constants.ts`)
   - Risk: Minimal - just adding constants
   - Mitigation: Follow naming conventions

2. **Input Reading** (`inputs.ts`)
   - Risk: Minimal - standard pattern
   - Mitigation: Use core.getInput() consistently

3. **Documentation**
   - Risk: None
   - Mitigation: N/A

### Breaking Changes
**None Expected** - All changes are additive

### Backward Compatibility
- ✅ Existing products unaffected
- ✅ New inputs are optional (only validate if enabled)
- ✅ No changes to existing outputs
- ✅ No changes to existing APIs

---

## Implementation Phases

### Phase 1: Foundation Setup (Est: 2-3 hours)
**Goal**: Set up basic structure without breaking anything

**Tasks**:
1. Add constants to `application-constants.ts`
   - Product key constants
   - Parameter key constants
   - Stage name
   - Input file name

2. Create input data model `input-data/[product].ts`
   - Define interfaces
   - Extend Common
   - Add JSDoc

3. Add inputs to `inputs.ts`
   - Read all new inputs
   - Export constants

4. Update `action.yml`
   - Add input definitions
   - Document each input

**Deliverables**:
- ✅ Constants defined
- ✅ Data model created
- ✅ Inputs readable
- ✅ Action.yml updated

**Testing**: None yet (no logic implemented)

**Risks**: LOW - No logic changes

---

### Phase 2: Validation Logic (Est: 2-3 hours)
**Goal**: Implement validation for new product

**Tasks**:
5. Create validator function in `validators.ts`
   - Implement validation logic
   - Follow existing pattern
   - Return error arrays

6. Write validator unit tests
   - Test all required inputs
   - Test optional inputs
   - Test custom validation rules
   - Test product disabled case

7. Update `bridge-cli.ts` validation
   - Add new product to validation check
   - Call new validator

**Deliverables**:
- ✅ Validator implemented
- ✅ Unit tests passing
- ✅ Integration with bridge-cli

**Testing**:
- Run unit tests: `npm test -- validators.test.ts`
- Verify all test cases pass

**Risks**: MEDIUM
- Validation logic might have edge cases
- Mitigation: Comprehensive unit tests

---

### Phase 3: Command Building (Est: 3-4 hours)
**Goal**: Build Bridge CLI commands for new product

**Tasks**:
8. Add command builder to `tools-parameter.ts`
   - Implement getFormattedCommandFor[Product]()
   - Implement get[Product]Data()
   - Generate JSON input file
   - Return CLI arguments

9. Write command builder unit tests
   - Test JSON generation
   - Test CLI argument formatting
   - Test optional field handling

10. Update `bridge-cli.ts` command preparation
    - Add product to prepareCommand()
    - Include in scan type detection

**Deliverables**:
- ✅ Command builder implemented
- ✅ JSON generation working
- ✅ CLI arguments correct

**Testing**:
- Run unit tests: `npm test -- tools-parameter.test.ts`
- Verify JSON structure
- Verify CLI arguments

**Risks**: MEDIUM
- Complex file (990+ lines)
- Mitigation: Follow existing pattern exactly

---

### Phase 4: Integration Testing (Est: 2-3 hours)
**Goal**: Test end-to-end workflow

**Tasks**:
11. Write contract/E2E tests
    - Test with all required inputs
    - Test with missing required inputs
    - Test with optional inputs
    - Test multi-product scenario

12. Update GitHub integration (if needed)
    - SARIF report handling
    - PR comment support
    - Artifact upload

13. Add utility functions (if needed)
    - SARIF path handling
    - Version detection

**Deliverables**:
- ✅ E2E tests passing
- ✅ GitHub integration working
- ✅ Full workflow validated

**Testing**:
- Run contract tests: `npm run contract-test`
- Test with actual Bridge CLI (if available)
- Verify SARIF generation and upload

**Risks**: MEDIUM
- E2E tests might reveal integration issues
- Mitigation: Test incrementally

---

### Phase 5: Documentation (Est: 1-2 hours)
**Goal**: Complete documentation

**Tasks**:
14. Update README.md
    - Add product section
    - Add usage examples
    - Add input documentation

15. Update CLAUDE.md
    - Document new product
    - Update architecture notes
    - Add to product list

16. Add inline documentation
    - JSDoc for all functions
    - Inline comments for complex logic

**Deliverables**:
- ✅ README updated
- ✅ CLAUDE.md updated
- ✅ Code documented

**Testing**: None

**Risks**: LOW

---

## Testing Strategy

### Unit Tests (Required)
1. Validator tests
   - All required inputs
   - Optional inputs
   - Custom validation rules
   - Product disabled

2. Command builder tests
   - JSON generation
   - CLI arguments
   - Optional fields

### Contract/E2E Tests (Required)
1. Full workflow with product enabled
2. Missing required inputs
3. Multi-product scenario
4. Error handling

### Manual Testing (Recommended)
1. Test with actual Bridge CLI
2. Verify SARIF generation
3. Test GitHub integration
4. Verify error messages

---

## Rollout Plan

### Step 1: Development
- Implement in feature branch
- Run all tests locally
- Code review

### Step 2: Testing
- Test with Bridge CLI (if available)
- Test in dev environment
- Verify all scenarios

### Step 3: Documentation
- Update all docs
- Add examples
- Review completeness

### Step 4: Release
- Merge to main
- Tag release
- Update release notes

---

## Potential Issues & Mitigation

### Issue 1: Bridge CLI Version Compatibility
- **Problem**: New product might require Bridge CLI version X.Y.Z+
- **Mitigation**: Document version requirement, add version check
- **Impact**: Users on old Bridge CLI won't support new product

### Issue 2: SARIF Format Changes
- **Problem**: New product might have different SARIF structure
- **Mitigation**: Test SARIF parsing, update path detection if needed
- **Impact**: GitHub Code Scanning upload might fail

### Issue 3: Validation Complexity
- **Problem**: Product has complex validation rules
- **Mitigation**: Break validation into smaller functions, add comprehensive tests
- **Impact**: More time needed for validation implementation

### Issue 4: Large tools-parameter.ts File
- **Problem**: File already 990+ lines, adding more increases complexity
- **Mitigation**: Follow existing pattern, consider refactoring in future
- **Impact**: Harder to maintain, risk of merge conflicts

---

## Success Criteria

### Functional Requirements
- ✅ New product can be configured via action inputs
- ✅ Validation prevents invalid configurations
- ✅ Bridge CLI command is correctly built
- ✅ SARIF reports are generated and uploaded
- ✅ Errors are properly handled and reported

### Quality Requirements
- ✅ All unit tests pass
- ✅ All E2E tests pass
- ✅ Code coverage >80% for new code
- ✅ No linting errors
- ✅ No TypeScript errors

### Documentation Requirements
- ✅ README includes usage examples
- ✅ CLAUDE.md documents architecture changes
- ✅ All code has JSDoc comments
- ✅ Inline comments for complex logic

---

## Checklist

### Before Starting
- [ ] Read existing product implementations (Polaris, Coverity, etc.)
- [ ] Understand Bridge CLI requirements for new product
- [ ] Identify all required inputs
- [ ] Review this implementation plan

### Phase 1: Foundation
- [ ] Constants added
- [ ] Data model created
- [ ] Inputs added
- [ ] action.yml updated

### Phase 2: Validation
- [ ] Validator function implemented
- [ ] Unit tests written and passing
- [ ] Bridge CLI integration updated

### Phase 3: Command Building
- [ ] Command builder implemented
- [ ] JSON generation working
- [ ] CLI arguments correct
- [ ] Unit tests passing

### Phase 4: Integration
- [ ] E2E tests written and passing
- [ ] GitHub integration working (if applicable)
- [ ] Full workflow tested

### Phase 5: Documentation
- [ ] README updated
- [ ] CLAUDE.md updated
- [ ] Code documented

### Before Release
- [ ] All tests pass
- [ ] Code reviewed
- [ ] Documentation complete
- [ ] Manual testing done

---

## Summary

**Total Estimated Time**: 10-15 hours
**Complexity**: MEDIUM
**Risk Level**: LOW-MEDIUM
**Breaking Changes**: NONE
**New Files**: 3
**Modified Files**: 6-8

**Key Points**:
- Follow existing product patterns exactly
- Extensive testing at each phase
- No breaking changes to existing products
- Backward compatible
- Well-documented

**Next Steps**:
1. Review this plan with team
2. Create feature branch
3. Start with Phase 1
4. Test incrementally
5. Document as you go
```

---

## Best Practices

### Analysis Principles
- **Comprehensive**: Map ALL affected files
- **Risk-aware**: Identify all potential risks
- **Phased**: Break into manageable phases
- **Testable**: Plan testing at each phase
- **Documented**: Document every decision

### When to Use
- Before implementing new features
- When planning major refactoring
- For impact analysis of changes
- Before releasing breaking changes

### Output Quality
- Specific file:line references
- Clear risk assessment
- Actionable implementation steps
- Comprehensive testing plan
- Realistic time estimates

---

## Example Usage

**User**: "Analyze what changes are needed to add support for a new product called Seeker"

**Response**:
1. Understand Seeker requirements
2. Map all affected files (10+ files)
3. Identify dependencies
4. Assess risks
5. Create phased implementation plan
6. Generate comprehensive document
7. Provide checklist and success criteria
