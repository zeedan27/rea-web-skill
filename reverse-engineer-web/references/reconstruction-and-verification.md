# Reconstruction and Verification Guide

This guide establishes the specification, implementation, and verification standards for clean-room reconstruction of web applications from reverse engineering Evidence.

---

## 1. Specification Templates

Reconstructions must be guided by unambiguous, evidenced specifications authored before writing implementation code.

### CLONE_SPEC.md Template

Every specification item must link to a verified Evidence ID (`evidence_id`) or file offset. Never copy proprietary source code, copyrighted assets, or private identifiers into the specification.

```markdown
# Application Specification: [App Name]

## 1. Target Identity & Provenance
- Target Name / URL: [Observed target]
- Binary / Archive SHA-256: [SHA-256 digest]
- Analysis Date & Toolchain: [e.g. REA 6.8.0]
- Root Evidence ID: [evidence_id]

## 2. Behavioral Workflows & State Machines
- State Machine: [e.g., Unauthenticated -> Authenticating -> Active -> Expired]
- User Workflows:
  - Workflow A: [Step-by-step observable user interaction]
    - Trigger: [Event / Button click]
    - State Transition: [State S1 -> State S2]
    - Observable Output: [DOM update / Network request]
    - Evidence ID: [ev_<sha256 from the tool result>]

## 3. Interfaces & Data Contracts
- Data Structures: [Entities, properties, types, optionality]
- Validation Rules: [Field constraints, regex patterns, limits]
- Evidence ID: [ev_<sha256 from the tool result>]

## 4. Algorithms & Business Formulas
- Computation: [Formula / business logic expressed in pseudo-code]
- Input Constraints: [Valid domains, rounding rules, overflow behavior]
- Evidence ID: [ev_<sha256 from the tool result>]

## 5. API Surface & Network Contracts
- Endpoint: [HTTP Method] [Path Template, e.g. /api/v1/projects/:id]
  - Headers: [Content-Type, custom headers]
  - Query Parameters: [Parameter names, types, defaults]
  - Request Body Schema: [JSON Schema or TypeScript interface]
  - Response Body Schema (Success 200/201): [JSON Schema]
  - Error Responses (400, 401, 404, 500): [Error payload structure]
  - Evidence ID: [ev_<sha256 from the tool result>]

## 6. Design Tokens & Presentation
- Color Palette: [Primary, secondary, neutral, error hex/RGB values]
- Typography: [Font families, scale, weights, line-heights]
- Spacing & Layout: [Scale in px/rem, breakpoints, container max-widths]
- Component Dimensions: [Button heights, border-radii, elevation shadows]
- Source: manual DevTools observation (date, page URL, viewport)

## 7. Residual Unknowns
- [Explicit list of unobserved behaviors, internal server logic, or unexercised branches]
- Unknown ID: [unknown_id]
```

### PARITY_CHECKLIST.md Template

Track verification coverage across web client-observable features:

```markdown
# Parity Verification Checklist

## Scope Tiers
- Must-Have: Core business logic, essential workflows, primary API contracts.
- Should-Have: Secondary error states, responsive view adjustments, keyboard navigation.
- Out-of-Scope: Proprietary analytics endpoints, third-party trackers, unexercised features.

| ID | Client-Observable Feature | Test Harness / Command | Expected Oracle Behavior | Status | Evidence ID |
| :--- | :--- | :--- | :--- | :--- | :--- |
| M-01 | Auth redirect on 401 | `npm test -- auth-flow.test.ts` | Redirect to /login?from=/dash | PASS | ev_<sha256 from the tool result> |
| M-02 | Route transition state | `npm test -- router.test.ts` | URL /items/:id renders view & history | PASS | ev_<sha256 from the tool result> |
| M-03 | Form client validation | `npm test -- form-val.test.ts` | Invalid email shows inline error | PASS | ev_<sha256 from the tool result> |
| S-01 | Offline asset caching | `npm test -- sw-cache.test.ts` | SW serves offline fallback shell | PASS | ev_<sha256 from the tool result> |
| S-02 | Responsive layout menu | `npm test -- responsive.test.ts` | Viewport <768px collapses sidebar | PASS | ev_<sha256 from the tool result> |
| S-03 | Bundle budget check | `npm run check:bundle-budget` | Initial vendor JS bundle <250 KB gzip | PASS | ev_<sha256 from the tool result> |

Status values: `PASS`, `FAIL`, `NOT-TESTED`.
Evidence ID format: `ev_<sha256 from the tool result>` (e.g. SHA-256 digest).
```

---

## 2. Three-Layer Rebuild Architecture

To ensure strict clean-room isolation and long-term maintainability, partition the reconstructed codebase into three decoupled layers:

```
[Layer 3: UI Presentation] -> Imports Layer 1 & 2; renders DOM trees & binds events.
         │
[Layer 2: Headless Logic]  -> Pure functions, state machines, API clients; ZERO UI imports.
         │
[Layer 1: Design Tokens]   -> Colors, typography, spacing (Source: manual DevTools, not REA).
```

1. **Layer 1: Design Tokens**: Pure declarations of visual constants (CSS custom properties, JSON theme files). Layer 1 tokens come from manual observation (read with browser DevTools or user-run script, not REA Evidence; REA does not extract CSS variables or design tokens).
2. **Layer 2: Headless Logic & State**: Pure TypeScript/JavaScript modules implementing domain models, state machines, business formulas, and API clients. This layer must have **zero UI dependencies** (no React/DOM imports), making it directly testable via automated differential harnesses.
3. **Layer 3: UI Presentation**: UI components that render visual layouts using Layer 1 tokens and bind interactions directly to Layer 2 state machines.

---

## 3. Differential Testing Patterns

Do not rely on informal spot-checks or self-attestation. Prove parity using differential testing against the original target as a black-box oracle:

### 1. Input Sweeps & Property Testing
Generate test input sweeps (edge numbers, unicode strings, empty collections, boundary timestamps) and execute them simultaneously against the original oracle and the reconstruction:

```typescript
// differential.test.ts
import { originalOracle } from "./oracleClient";
import { rebuiltModule } from "../src/pureLogic";

test.each(edgeCaseInputs)("parity on input %p", async (input) => {
  const oracleResult = await originalOracle.compute(input);
  const rebuildResult = rebuiltModule.compute(input);
  expect(rebuildResult).toEqual(oracleResult);
});
```

### 2. Golden Fixtures
Record responses and calculated states from the original application during Phase 4 runtime observation. Save these as versioned JSON fixtures and assert that the reconstruction produces matching results.

### 3. Visual and DOM Parity
Capture screenshots of original pages and reconstructed pages under identical viewport dimensions. Use `compare_web_screenshots` (`src/contracts/browserToolContracts.ts:313`) to measure pixel-level deltas:
- Takes `before` and `after` PNG file paths and an optional `channel_threshold`.
- Returns match metrics, total pixel differences, and bounding boxes without uploading images to third-party services.

### 4. HAR Replay Against Mock Backends
Replay recorded network requests (`inspect_web_network_capture`, using `capture_path` and `format`) against mock servers implementing reconstructed API schemas. Verify that the client handles all response structures and error statuses identically to the original.

---

## 4. Rebuild Guardrails

1. **No Spot-Check Attestation**: Never declare functional equivalence after observing a handful of manual actions. Parity requires automated test suites covering happy paths, edge cases, and failure modes.
2. **No One-Shot Generation**: Avoid issuing large "rebuild the whole application" prompts. Implement one small verified slice at a time:
   `Tokens -> Domain Models -> Business Logic -> API Client -> UI Component -> Integration Test`.
3. **Treat Model Output as Untrusted**: The language model's code output and explanatory claims must be treated as untrusted hypotheses until automated tests execute against the test oracle and pass.
4. **Preserve Unknowns**: Where original behavior could not be observed (e.g. unexercised server validation), document it as an explicit Unknown rather than inventing behavior.

---

## 5. Verification Tool Contract: `verify_reconstruction`

REA provides a formal verification contract: `verify_reconstruction` (MCP Tool contract declared via `session("verify_reconstruction", ...)` in `src/contracts/sessionToolContracts.ts:150`; distinct from the MCP prompt at `promptContracts.ts:314`; domain: `src/domain/reconstructionVerification.ts`).

### Schema Definitions and Citations
- **Evidence ID Format**: `src/domain/reconstructionVerificationSchemas.ts:8` defines `evidenceIdSchema = prefixedDigestSchema("ev")`. All Evidence IDs must match `ev_<sha256 from the tool result>` (lowercase hex SHA-256 digest prefixed with `ev_`).
- **Claim Kinds**: `src/domain/reconstructionVerificationSchemas.ts:20` defines `claimKindSchema = z.enum(["behavioral", "structural-artifact", "structural-function"])`.
- **Claim Dimensions**: `src/domain/reconstructionVerificationSchemas.ts:26-36` defines `claimDimensionSchema = z.enum(["overall", "calls", "exports", "imports", "metadata", "sections"])`.
- **Comparison Operations**: `src/domain/reconstructionVerification.ts:36-40` binds claim kinds to specific comparison operations:
  ```typescript
  const comparisonOperationsByClaimKind: Record<ClaimKind, ComparisonOperation> = {
    behavioral: "compare_process_captures",
    "structural-function": "compare_functions",
    "structural-artifact": "compare_artifacts",
  };
  ```

### Tool Purpose and Arguments
`verify_reconstruction` evaluates whether declared reconstruction claims are supported by retained comparison Evidence:
- **Input Schema** (`src/domain/reconstructionVerificationSchemas.ts:66`):
  `specification`: Object containing `name` (string) and `claims` (array of claim objects):
    - `claim_id`: Unique string identifier for the claim.
    - `kind`: `"behavioral"` | `"structural-artifact"` | `"structural-function"`.
    - `dimension`: Claim aspect (`"overall"`, `"calls"`, `"exports"`, `"imports"`, `"metadata"`, `"sections"`).
    - `comparison_evidence_id`: Evidence ID (`ev_<sha256 from the tool result>`) of a retained comparison operation (`compare_process_captures`, `compare_artifacts`, or `compare_functions`).
    - `source_evidence_ids`: Evidence IDs (`ev_<sha256 from the tool result>`) of the original and reconstructed source observations.

### Multi-Source Process Capture Audit (Source Facts)
Inspection of `src/domain/reconstructionVerification.ts` establishes how multiple process-capture sources are handled:
- Lines 193–217: Validates that every source in `sources` has `source.operation === "capture_process_scenario"` and `predicate_type === "rea.process-capture"`. Throws `EvidenceIntegrityError` if any source differs.
- Lines 219–226: Iterates over every source (`for (const source of sources)`) and parses each with `readEvidencePayload(processCaptureSchema, source.normalized_result)`.
- Lines 368–383: Verifies that no source appears on both `sides.left` and `sides.right`, and validates that every source has `source.authority === "controlled-replay"` and `source.confidence === "observed"`.
- Lines 389–403: Replay environment validation uses `sources[0]?.environment` as the baseline against which all other sources are checked.
- **Third-Party Claim Assessment**: The claim that "only the first source is checked" is **REFUTED** in 6.8.0. Every source is fully type-checked, schema-validated, and authority-verified.

### Critical Rule: One Signal, Never the Only One
Use `verify_reconstruction` as **ONE signal, never the only one**:
- **Scope Limitation**: The tool verifies only the specific claims declared in the input array. Line 109 explicitly declares:
  > *"Pass means every declared claim passed; it does not establish global implementation equivalence."*
- **Independent Verification Required**: Supplement tool checks with end-to-end integration tests, user workflow simulations, and differential test suites.
