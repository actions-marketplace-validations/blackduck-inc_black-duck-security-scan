---
name: code-explainer
description: Comprehensive code explanation skill that analyzes and explains the Black Duck Security Scan codebase architecture, execution flow, data flow, design patterns, and component relationships. Use when users ask "how does this work", "explain the architecture", "trace the execution", or "what patterns are used".
---

# Code Explainer

Provides comprehensive analysis and explanation of the Black Duck Security Scan GitHub Action codebase, covering architecture, execution flow, data transformations, and design patterns.

## Usage

Run this skill when the user asks questions like:
- "Explain how this codebase works"
- "What is the architecture of this project?"
- "How does the data flow through the system?"
- "What happens when the action runs?"
- "Trace the execution from start to finish"
- "What design patterns are used?"
- "How do the components interact?"
- "Show me the module structure"
- "How are errors handled end-to-end?"

## Analysis Modes

This skill operates in different modes based on the user's question:

### Mode 1: Architecture Analysis
For questions about structure, organization, and components.

### Mode 2: Flow Analysis
For questions about execution, data flow, and control flow.

### Mode 3: Pattern Analysis
For questions about design patterns and architectural decisions.

### Mode 4: Comprehensive Analysis
For general "explain how this works" questions.

---

## Mode 1: Architecture Analysis

### Step 1 — High-Level Architecture Overview

Provide a clear overview of:

1. **Project Type & Purpose**
   - GitHub Action for integrating Black Duck security testing (SAST/SCA)
   - Supports Polaris, Coverity, Black Duck SCA, and SRM products
   - Orchestrates Bridge CLI for multi-product scanning

2. **Entry Points**
   - Main entry: `src/main.ts:run()` (line 24)
   - Workflow: Initialize → Validate → Execute → Report → Cleanup

3. **Core Components**
   - **Bridge CLI Manager** (`bridge-cli.ts`): Downloads and executes Bridge CLI
   - **Input Layer** (`inputs.ts`): Reads GitHub Action inputs from action.yml
   - **Validation Layer** (`validators.ts`): Validates product-specific inputs
   - **Input Data Models** (`input-data/`): TypeScript interfaces for each product
   - **Command Builder** (`tools-parameter.ts`): Constructs Bridge CLI commands
   - **GitHub Services** (`service/`): SARIF upload and artifact management
   - **Factory** (`factory/`): Runtime service selection (Cloud vs Enterprise)
   - **Utilities** (`utility.ts`): Helper functions for paths, versions, etc.

4. **External Dependencies**
   - `@actions/core`, `@actions/exec`, `@actions/artifact`: GitHub Actions SDK
   - `@actions/tool-cache`: Caching downloaded tools
   - `@actions/github`: GitHub API client (Octokit)
   - Bridge CLI: Black Duck's security testing orchestrator

### Step 2 — Module Organization

```
src/
├── main.ts                              # Entry point (24 lines to run())
├── application-constants.ts             # Centralized string constants
└── blackduck-security-action/
    ├── inputs.ts                        # Input collection from action.yml
    ├── validators.ts                    # Multi-product input validation
    ├── bridge-cli.ts                    # Bridge CLI orchestration
    ├── tools-parameter.ts               # Command builder (990+ lines)
    ├── artifacts.ts                     # Diagnostics & SARIF upload
    ├── utility.ts                       # Helper functions (350+ lines)
    ├── factory/
    │   └── github-client-service-factory.ts  # Runtime service selection
    ├── service/
    │   ├── github-client-service-interface.ts
    │   └── impl/
    │       ├── github-client-service-base.ts      # Template methods
    │       ├── github-client-service-cloud.ts     # GitHub Cloud impl
    │       └── github-client-service-v1.ts        # Enterprise impl
    └── input-data/
        ├── common.ts                    # Shared interfaces (Network, Bridge)
        ├── polaris.ts                   # Polaris data model
        ├── coverity.ts                  # Coverity data model
        ├── blackduck.ts                 # Black Duck SCA data model
        └── srm.ts                       # SRM data model
```

**Module Responsibilities**:

- **main.ts**: Orchestrates entire action lifecycle with cleanup in finally block
- **inputs.ts**: Single source of truth for action inputs, handles deprecation
- **validators.ts**: Product-specific validation, returns string[] errors
- **bridge-cli.ts**: Manages Bridge CLI download, validation, command execution
- **tools-parameter.ts**: Builds product-specific JSON config files and CLI args
- **service/impl/**: Handles SARIF upload with retry logic for different GitHub types
- **factory/**: Detects GitHub Cloud vs Enterprise Server and returns appropriate service
- **input-data/**: TypeScript interfaces matching Bridge CLI JSON input schema

### Step 3 — Design Patterns Used

#### 1. Factory Pattern
- **Location**: `factory/github-client-service-factory.ts:11-67`
- **Purpose**: Runtime selection between GitHub Cloud vs Enterprise Server
- **Implementation**: Static factory method returning IGithubClientService
- **Example Usage**: `main.ts:92-113` - Upload SARIF to appropriate GitHub type
- **Benefits**: Abstraction, easy extensibility for new GitHub versions

#### 2. Builder Pattern (Implicit)
- **Location**: `tools-parameter.ts` methods
- **Purpose**: Constructs complex Bridge CLI commands with JSON inputs
- **Implementation**: Methods like `getFormattedCommandForPolaris()` (line 36-355)
- **Process**: Parse inputs → Build JSON → Write file → Return CLI args
- **Benefits**: Separates command construction from execution

#### 3. Strategy Pattern
- **Location**: `service/impl/` directory
- **Purpose**: Different SARIF upload strategies for Cloud vs Enterprise
- **Implementation**: Multiple implementations of IGithubClientService interface
- **Strategies**:
  - `GithubClientServiceCloud`: Uses Code Scanning API
  - `GithubClientServiceV1`: Uses Enterprise API v1
- **Benefits**: Swappable implementations, testability

#### 4. Template Method Pattern
- **Location**: `service/impl/github-client-service-base.ts`
- **Purpose**: Common retry logic with customizable implementations
- **Implementation**: Base class with abstract methods for subclasses
- **Template**: Retry wrapper → Call subclass method → Handle errors
- **Benefits**: DRY principle, consistent error handling

#### 5. Adapter Pattern (Partial)
- **Location**: `artifacts.ts`
- **Purpose**: Wrapper for @actions/artifact v1 and v2 APIs
- **Implementation**: Conditional logic based on available API version
- **Benefits**: Backward compatibility with different artifact API versions

### Step 4 — Data Flow Architecture

```
GitHub Action Inputs (action.yml)
  ↓
Input Collection (inputs.ts)
  ↓
Validation (validators.ts)
  ↓
Data Model Construction (input-data/*.ts)
  ↓
JSON File Generation (tools-parameter.ts)
  ↓
Bridge CLI Execution (bridge-cli.ts)
  ↓
SARIF Report Generation (by Bridge CLI)
  ↓
GitHub Upload (service/impl/*.ts)
  ↓
Artifact Upload (artifacts.ts)
```

**Data Transformation Layers**:
1. **String inputs** (from action.yml) → **TypeScript constants** (inputs.ts)
2. **Constants** → **Validated data** (validators.ts checks)
3. **Validated data** → **TypeScript interfaces** (input-data/*.ts)
4. **Interfaces** → **JSON files** (tools-parameter.ts writes to temp)
5. **JSON files** → **CLI arguments** (--input flags)
6. **Bridge CLI output** → **SARIF files** (in temp directory)
7. **SARIF files** → **GitHub Code Scanning** (via API)

### Step 5 — Key Architectural Decisions

#### 1. Product Abstraction
- **Decision**: Separate input data models for each product
- **Rationale**: Products have different configuration schemas
- **Trade-offs**: More files but better type safety and clarity
- **Implementation**: `input-data/polaris.ts`, `input-data/coverity.ts`, etc.

#### 2. Version Compatibility
- **Decision**: Conditional logic based on Bridge CLI version
- **Rationale**: Bridge CLI 2.0.0+ changed SARIF output paths
- **Implementation**: `utility.ts:updatePolarisSarifPath()` checks version
- **Challenge**: Version detection via directory structure or metadata

#### 3. Platform Support
- **Decision**: Platform-specific Bridge CLI binaries
- **Rationale**: Bridge CLI has different downloads for Windows, macOS, Linux, ARM
- **Implementation**: `bridge-cli.ts` detects platform and downloads appropriate binary
- **Minimum versions**: ARM support requires Bridge CLI 2.3.0+ (Linux) or 2.4.0+ (macOS)

#### 4. Error Handling Strategy
- **Decision**: Accumulate validation errors, don't fail fast
- **Rationale**: Show all validation issues at once for better UX
- **Implementation**: validators return string[] which are joined and thrown together
- **Exit codes**: Mapped in `application-constants.ts` to human-readable messages

#### 5. Network & Air-Gap Mode
- **Decision**: Support both download mode and local installation
- **Rationale**: Some environments don't allow external downloads
- **Implementation**: `network_airgap` input skips download, uses `bridgecli_install_directory`
- **Trade-off**: Users must pre-install Bridge CLI in air-gap mode

---

## Mode 2: Flow Analysis

### Main Execution Flow

```
GitHub Action Triggered
  ↓
main.ts:run() (line 24)
  ↓
1. Create temp directory (lines 25-30)
  ↓
2. Instantiate Bridge class (lines 35-37)
   ↓ bridge-cli.ts constructor (line 47)
   ↓ validateBridgePath() (line 78-106)
   ├─ Check network_airgap mode
   ├─ Download Bridge CLI if needed (download-utility.ts)
   └─ Validate local installation
  ↓
3. Prepare commands (bridge.prepareCommand())
   ↓ Validate scan types (line 200-240)
   ↓ For each enabled product:
   │  ├─ validatePolarisInputs()
   │  ├─ validateCoverityInputs()
   │  ├─ validateBlackDuckInputs()
   │  └─ validateSRMInputs()
   ↓ Build commands (tools-parameter.ts)
   │  ├─ getFormattedCommandForPolaris()
   │  ├─ getFormattedCommandForCoverity()
   │  ├─ getFormattedCommandForBlackduck()
   │  └─ getFormattedCommandForSRM()
   ↓ Return formatted command array
  ↓
4. Execute Bridge CLI (line 252)
   ↓ exec.exec('bridge-cli', args, options)
   ↓ Capture exit code
  ↓
5. Process exit code (main.ts:39-90)
   ├─ 0 = Success
   ├─ 8 = Policy violation (conditional fail)
   └─ Other = Failure
  ↓
6. Upload SARIF reports (if exit code 0 or 8)
   ↓ GitHubClientServiceFactory.getInstance()
   ↓ service.uploadSarifReport(sarifPath)
  ↓
7. Upload diagnostics (if include_diagnostics enabled)
   ↓ artifacts.ts:uploadDiagnostics()
  ↓
8. Cleanup temp directory (finally block)
  ↓
9. Set build status based on exit code
```

### Data Flow Trace

#### Phase 1: Input Collection (`inputs.ts`)
```typescript
action.yml inputs
  ↓
core.getInput('polaris_server_url')
  ↓
export const POLARIS_SERVER_URL = core.getInput('polaris_server_url')
  ↓
Used in validators and tools-parameter
```

#### Phase 2: Input Transformation (`tools-parameter.ts:36-355`)
```typescript
Input: POLARIS_SERVER_URL, POLARIS_ACCESS_TOKEN, etc.
  ↓
Parse and validate
  ├─ Split comma-separated values (e.g., assessment types)
  ├─ Parse booleans (e.g., enable_pull_request_comment)
  └─ Resolve file paths (e.g., polaris_application_name_file)
  ↓
Build JSON Structure
{
  data: {
    polaris: {
      accesstoken: '***',
      serverUrl: 'https://polaris.synopsys.com',
      application: {name: 'MyApp'},
      project: {name: 'MyProject'},
      assessment: {types: ['SAST', 'SCA']}
    },
    network: {...},
    bridge: {...}
  }
}
  ↓
Write to temp directory
fs.writeFileSync('/tmp/polaris_input.json', JSON.stringify(polarisData))
  ↓
Return CLI arguments
['--stage', 'polaris', '--input', '/tmp/polaris_input.json']
```

#### Phase 3: Execution Output Processing
```typescript
Bridge CLI execution
  ↓
Generates SARIF files
  ├─ Bridge CLI v1.x: 'Polaris SARIF Generator/results.sarif.json'
  └─ Bridge CLI v2.0+: 'integrations/polaris/sarif/results.sarif.json'
  ↓
Update paths if diagnostics enabled (utility.ts)
  ↓
Upload to GitHub Code Scanning
  ├─ Read SARIF file
  ├─ Get commit SHA, ref, repository info
  └─ POST to /repos/{owner}/{repo}/code-scanning/sarifs
  ↓
Upload diagnostics as artifact
  └─ artifacts.ts: zip diagnostics → upload via @actions/artifact
```

### Control Flow Decision Points

#### Decision 1: Product Selection
```
Check: POLARIS_SERVER_URL set?
  ├─ YES → validatePolarisInputs()
  │         ├─ Valid → Build Polaris command
  │         └─ Invalid → Add errors to array
  └─ NO → Skip Polaris

Check: COVERITY_URL set?
  └─ [Same pattern]

Any commands built?
  ├─ YES → Execute Bridge CLI
  └─ NO → Throw validation error
```

#### Decision 2: Version Compatibility
```
Detect Bridge CLI version (from directory structure or metadata)
  ↓
Compare with threshold (2.0.0)
  ├─ Version < 2.0.0 → Use old SARIF path
  └─ Version >= 2.0.0 → Use new SARIF path
```

#### Decision 3: GitHub Service Selection
```
Get GITHUB_SERVER_URL from environment
  ↓
Is GitHub Cloud (github.com)?
  ├─ YES → Return GithubClientServiceCloud
  └─ NO → Detect Enterprise version
            ├─ Query /api/v3/meta endpoint
            ├─ Parse installed_version
            └─ Return GithubClientServiceV1
```

### Error Propagation Flow

#### Validation Errors
```
validators.ts:validatePolarisInputs() → Returns string[]
  ↓
bridge-cli.ts:prepareCommand() → Accumulates errors
  ↓
if (formattedCommand.length === 0)
  ↓
throw new Error(validationErrors.join(','))
  ↓
main.ts:catch block
  ↓
Log error → Upload diagnostics → setFailed()
```

#### Execution Errors
```
exec.exec('bridge-cli', args) → Throws on non-zero exit
  ↓
main.ts:catch block
  ↓
getBridgeExitCode(error) → Extract exit code from message
  ↓
logBridgeExitCodes(error.message) → Map to human-readable message
  ↓
Check if exit code is 8 (policy violation)
  ├─ mark_build_status = false → Allow workflow to continue
  └─ mark_build_status = true → Fail workflow
  ↓
Upload diagnostics if enabled
  ↓
setFailed(error.message)
```

#### Network Errors (with Retry)
```
download-utility.ts:downloadFile() → HTTP request
  ↓
Wrapped in retry-helper.ts:execute()
  ↓
HTTP request fails (e.g., 503)
  ↓
isRetryable(error)? → Check HTTP status code
  ├─ Retryable (503, 408, 429, 500, 502, 504)
  │   ├─ Wait exponentially (15s → 30s → 60s)
  │   └─ Retry up to 3 times
  └─ Non-retryable (400, 401, 403, 404, 416, 422)
      └─ Throw immediately without retry
```

---

## Mode 3: Pattern Analysis

### Patterns in Use

#### Factory Pattern ⭐
- **File**: `factory/github-client-service-factory.ts`
- **Lines**: 11-67
- **Type**: Static Factory Method
- **Creates**: IGithubClientService implementations
- **Usage**: `main.ts:92` - Runtime service selection based on GitHub URL
- **Benefits**: Abstraction, extensibility, centralized version detection

#### Builder Pattern (Implicit) 🔨
- **File**: `tools-parameter.ts`
- **Type**: Command Builder
- **Builds**: Bridge CLI commands with JSON input files
- **Methods**: `getFormattedCommandForPolaris()`, etc.
- **Process**: Parse → Validate → Build JSON → Write file → Return args
- **Opportunity**: Could refactor to fluent builder for better composition

#### Strategy Pattern 🎯
- **File**: `service/impl/` directory
- **Interface**: IGithubClientService
- **Implementations**:
  - `GithubClientServiceCloud` - GitHub.com SARIF upload
  - `GithubClientServiceV1` - Enterprise Server v1 upload
- **Selection**: Factory pattern chooses strategy at runtime
- **Benefits**: Swappable, testable, follows Open/Closed principle

#### Template Method Pattern 📋
- **File**: `service/impl/github-client-service-base.ts`
- **Type**: Abstract base class with template methods
- **Template**: `uploadWithRetry()` - Retry logic wrapper
- **Customization**: Subclasses implement specific upload logic
- **Benefits**: DRY, consistent error handling and retry behavior

#### Adapter Pattern 🔌
- **File**: `artifacts.ts`
- **Type**: API version adapter
- **Adapts**: @actions/artifact v1 and v2 APIs
- **Implementation**: Conditional logic based on available API
- **Benefits**: Backward compatibility across GitHub Actions versions

### Pattern Opportunities

#### 1. Fluent Builder for Commands
**Current**: `tools-parameter.ts` methods with nested conditionals (990+ lines)
**Problem**: Hard to test, complex conditional logic, poor readability
**Proposed**:
```typescript
new BridgeCommandBuilder()
  .forPolaris()
    .withServerUrl(url)
    .withAccessToken(token)
    .withAssessmentTypes(['SAST', 'SCA'])
    .enablePRComments()
  .forBlackDuck()
    .withUrl(url)
    .withApiToken(token)
  .build()
```
**Benefits**: Better composition, testability, readability

#### 2. Validation Strategy Pattern
**Current**: Product-specific validation functions in `validators.ts`
**Problem**: Code duplication, hard to add new products
**Proposed**:
```typescript
interface ValidationStrategy {
  validate(inputs: Map<string, string>): string[]
}
class PolarisValidationStrategy implements ValidationStrategy {...}
class CoverityValidationStrategy implements ValidationStrategy {...}
```
**Benefits**: Easier to add products, better testability

#### 3. Command Pattern for Execution
**Current**: Direct command building and execution
**Problem**: Difficult to log, test, or undo operations
**Proposed**:
```typescript
interface BridgeCommand {
  execute(): Promise<number>
  undo(): Promise<void>
}
class PolarisScanCommand implements BridgeCommand {...}
```
**Benefits**: Better logging, undo capability, command queueing

### Anti-Patterns Detected

#### 1. God Object
- **Location**: `tools-parameter.ts` (990+ lines)
- **Issue**: Single class handles all product command building
- **Impact**: Hard to test, maintain, extend
- **Recommendation**: Extract product-specific builders:
  - `PolarisCommandBuilder`
  - `CoverityCommandBuilder`
  - `BlackDuckCommandBuilder`
  - `SRMCommandBuilder`

#### 2. Large Class / Utility Bag
- **Location**: `utility.ts` (350+ lines, 35+ functions)
- **Issue**: Lacks cohesion, mixed responsibilities (paths, versions, SARIF, SSO)
- **Impact**: Unclear purpose, difficult to navigate
- **Recommendation**: Split into focused modules:
  - `path-utils.ts` - Path manipulation
  - `version-utils.ts` - Version comparison
  - `sarif-utils.ts` - SARIF path updates
  - `sso-utils.ts` - SSO URL handling

---

## Mode 4: Comprehensive Analysis

When user asks general "explain how this works" questions, provide all three modes:

1. **Architecture Overview** (from Mode 1)
   - High-level components and their responsibilities
   - Module organization

2. **Execution Flow** (from Mode 2)
   - Step-by-step trace from entry to exit
   - Data transformations at each layer

3. **Pattern Summary** (from Mode 3)
   - Key patterns used and why
   - Architectural decisions and trade-offs

---

## Output Format

```markdown
# Code Explanation: [Topic]

## Architecture

[Component diagram or description]

### Key Components
- **Component Name** (`file.ts:lines`): [Purpose]

## Execution Flow

[Step-by-step trace with file:line references]

## Design Patterns

### Pattern Name
- **Location**: `file.ts:lines`
- **Purpose**: [Why used]
- **Benefits**: [What it provides]

## Key Takeaways

- [Important architectural decisions]
- [How to extend or modify]
```

## Best Practices

- **Use file:line references**: Always include exact locations
- **Show relationships**: Explain how components interact
- **Explain WHY**: Not just what code does, but why it's structured that way
- **Include trade-offs**: Discuss pros/cons of architectural decisions
- **Visual aids**: Use diagrams, flow charts, decision trees
- **Real examples**: Reference actual usage from the codebase

## Example Questions Handled

**Q: "How does this codebase work?"**
→ Provide Mode 4 (Comprehensive Analysis)

**Q: "Explain the architecture"**
→ Provide Mode 1 (Architecture Analysis)

**Q: "What happens when I run a Polaris scan?"**
→ Provide Mode 2 (Flow Analysis) with Polaris-specific trace

**Q: "What design patterns are used?"**
→ Provide Mode 3 (Pattern Analysis)

**Q: "How does SARIF upload work?"**
→ Combination of Flow + Pattern (Factory + Strategy patterns, upload flow)
