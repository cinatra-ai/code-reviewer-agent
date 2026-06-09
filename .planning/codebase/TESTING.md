# Testing Patterns

**Analysis Date:** 2026-06-09

## Test Framework

**Runner:**
- Not explicitly configured in `package.json` (no `test` script, no `jest.config.*`, no `vitest.config.*`). The CI workflow runs `corepack pnpm test --if-present` and silently skips if absent.

**Assertion Library:**
- Not detected.

**Run Commands:**
```bash
# No test runner configured — CI skips gracefully:
corepack pnpm test --if-present
```

## Test File Organization

**Location:** No test files exist in this repo. This is a content-only extracted agent repo — the `cinatra/oas.json` OAS body and `skills/code-review-methodology/SKILL.md` are the primary artifacts.

**Naming:** Not applicable — no test files present.

## Test Structure

**Note:** The repo's primary testable logic lives in `extension-kind-gate.mjs`, which exports all its validator functions as named exports (`validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`, `parseArgs`). This design enables unit testing without invoking `main()` or spawning a child process.

No test suite is currently wired. In the monorepo from which this repo is extracted, tests would live alongside the source and run in the monorepo's test pipeline.

## Mocking

**Framework:** Not applicable — no test framework configured.

**What to Mock (guidance from gate design):**
- File system calls (`readFileSync`, `existsSync`, `readdirSync`) — these are the only side effects in `extension-kind-gate.mjs`. Tests should mock `node:fs` to avoid fixture files on disk.
- `process.argv` / `process.exit` — for `parseArgs` and `main()` boundary tests.

**What NOT to Mock:**
- The pure validator functions (`validateBpmnSanity`, `validateWorkflowPackageShape`, `walkLlmStrings`, `scanOasString`) — these take plain strings/objects and return `string[]`. They should be tested with inline fixtures, not mocks.

## Fixtures and Factories

**Test Data:** No fixture files present. The validators accept inline strings:
- `validateBpmnSanity(xml: string)` — pass inline XML strings
- `validateWorkflowPackageShape(pkg: object)` — pass inline JS objects
- `validateAgent(packageRoot)` — requires a directory with `cinatra/oas.json`

**Location:** Not applicable — no fixtures present.

## Coverage

**Requirements:** None enforced — no coverage config or thresholds detected.

**View Coverage:** Not configured.

## Test Types

**Unit Tests:**
- Intended scope: pure functions in `extension-kind-gate.mjs` — `parseArgs`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `walkLlmStrings`, `scanOasString`, `wordBoundary`. All are stateless and return deterministic outputs.

**Integration Tests:**
- Intended scope: `validateAgent(packageRoot)` and `validateWorkflow(packageRoot)` against temp directory fixtures containing `package.json` + `cinatra/oas.json` or `cinatra/workflow.bpmn`.

**E2E Tests:**
- Not used. CI acts as the E2E gate: `node extension-kind-gate.mjs --package-root .` runs against the actual repo artifacts on every push/PR.

## CI Gate as a Substitute for Tests

The primary quality gate for this repo is the CI pipeline (`.github/workflows/ci.yml`), which runs:
1. **Dependency shape validation** — inline Node script checks that no `@cinatra-ai/*` packages leaked into `dependencies`/`devDependencies`.
2. **Typecheck** — skipped for this repo (source mirror with host-internal peers declared; the monorepo typechecks).
3. **Test** — skipped (`--if-present`, no test script).
4. **Pack dry-run** — `npm pack --dry-run` validates package shape and publish payload.
5. **Agent OAS gate** — `node extension-kind-gate.mjs --package-root .` scans `cinatra/oas.json` for retired CRM primitives in LLM-visible fields.

The `kind-gates` job (`.github/workflows/ci.yml` L129–146) runs after `build` and executes the agent-specific OAS validation gate.

## Common Patterns

**Testing pure validators (recommended pattern for future tests):**
```js
import { validateBpmnSanity } from "./extension-kind-gate.mjs";

// Pass: minimal valid BPMN
const xml = `<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL">
  <bpmn:process id="p1" /></bpmn:definitions>`;
const errors = validateBpmnSanity(xml);
assert.deepEqual(errors, []);

// Fail: missing BPMN namespace
const badXml = `<definitions><process /></definitions>`;
const badErrors = validateBpmnSanity(badXml);
assert.ok(badErrors.some(e => e.includes("BPMN 2.0 MODEL namespace")));
```

**Testing OAS banned-primitive scan:**
```js
import { validateAgent } from "./extension-kind-gate.mjs";
// Requires a temp dir with cinatra/oas.json containing
// a system/user/description field with a banned token like "lists_list"
```

---

*Testing analysis: 2026-06-09*
