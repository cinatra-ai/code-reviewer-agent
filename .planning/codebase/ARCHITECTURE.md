<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌─────────────────────────────────────────────────────────────────┐
│                  Cinatra OAS Flow 26.1.0 Runtime                │
│   (caller: /chat agent authoring orchestrator)                  │
└────────────────────────┬────────────────────────────────────────┘
                         │ invokes Flow via agent_run_id
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              StartNode (id: start)                               │
│   Inputs: oasJson, packageSlug, reviewContext, agent_run_id     │
│   `cinatra/oas.json` → $referenced_components.start             │
└────────────────────────┬────────────────────────────────────────┘
                         │ DataFlowEdges
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              ApiNode (id: review)                                │
│   POST {{CINATRA_BASE_URL}}/api/llm-bridge                      │
│   Model: openai / gpt-5.5                                       │
│   Skill: `skills/code-review-methodology/SKILL.md`              │
│   `cinatra/oas.json` → $referenced_components.review            │
└────────────────────────┬────────────────────────────────────────┘
                         │ DataFlowEdge: findings (string)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              EndNode (id: end)                                   │
│   Output: findings (JSON string)                                 │
│   `cinatra/oas.json` → $referenced_components.end               │
└─────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| StartNode (`start`) | Accept inputs; expose `oasJson`, `packageSlug`, `reviewContext`, `agent_run_id` | `cinatra/oas.json` |
| ApiNode (`review`) | Call LLM bridge with system/user prompts; apply code-review-methodology skill | `cinatra/oas.json` |
| EndNode (`end`) | Surface `findings` JSON string to the caller | `cinatra/oas.json` |
| Skill definition | Codify all review checks; instruct the LLM what to check and how to respond | `skills/code-review-methodology/SKILL.md` |
| CI / extension-kind gate | Validate OAS parses + no retired CRM primitives in LLM-visible prompt strings | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Cinatra OAS Flow 26.1.0 — a linear, three-node DAG agent (StartNode → ApiNode → EndNode) with a single LLM call.

**Key Characteristics:**
- Pure advisory: no blockers emitted; deterministic lint owns hard failures
- Stateless: no MCP calls, no state store, no side effects — pure shape inspection
- Skill-injected reasoning: `code-review-methodology` SKILL.md is delivered to the LLM as a system-level instruction
- Output envelope contract: LLM MUST return `{"findings":[...]}` (not a bare array) because WayFlow's DataFlowEdge extracts the `findings` key

## Layers

**Flow definition layer:**
- Purpose: Declare the agent's nodes, control flow, and data flow in OAS JSON
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNode, EndNode, ControlFlowEdges, DataFlowEdges, $referenced_components
- Depends on: Cinatra WayFlow runtime (`agentspec_version: 26.1.0`)
- Used by: WayFlow runtime at invocation time

**Skill/reasoning layer:**
- Purpose: Encode the code-review checks and LLM output contract as a structured skill
- Location: `skills/code-review-methodology/SKILL.md`
- Contains: Input/output spec, check categories (naming, version hygiene, metadata, OAS shape, HITL hints), step-by-step instructions, exclusion list
- Depends on: Nothing (pure text delivered inline to the LLM)
- Used by: ApiNode `review` via the Cinatra skill injection mechanism

**CI gate layer:**
- Purpose: Zero-dependency pre-publish sanity check — parses OAS, scans for retired CRM primitives in LLM-visible fields
- Location: `extension-kind-gate.mjs`
- Contains: `validateAgent`, `runGate`, `walkLlmStrings`, `scanOasString`, `validateWorkflow`, `validateBpmnSanity`, `findWorkflowSidecars`
- Depends on: Node.js builtins only (`fs`, `path`) — intentionally no `@cinatra-ai/*` dependency
- Used by: `.github/workflows/ci.yml` (`kind-gates` job)

**Package manifest layer:**
- Purpose: Identify the extension to the Cinatra registry and declare its kind
- Location: `package.json`
- Contains: `cinatra.apiVersion`, `cinatra.kind: "agent"`, `cinatra.dependencies`, `name`, `version`, `description`
- Depends on: Nothing at runtime
- Used by: Cinatra marketplace publish/install, CI dependency-shape check

## Data Flow

### Primary Request Path

1. Caller (orchestrator) invokes the agent, passing `oasJson` (required), `packageSlug`, `reviewContext`, `agent_run_id` (`cinatra/oas.json` → `inputs`)
2. StartNode forwards all four inputs via DataFlowEdges to the ApiNode (`cinatra/oas.json` → `data_flow_connections`)
3. ApiNode `review` posts to `/api/llm-bridge` with a system prompt (referencing the skill), a user prompt interpolating `{{ packageSlug }}`, `{{ reviewContext }}`, `{{ oasJson }}`, and `cinatra_llm` config (`cinatra/oas.json` → `$referenced_components.review.data`)
4. LLM bridge returns `{"findings":[...]}` as a string
5. DataFlowEdge `review_to_end_findings` routes the `findings` output to EndNode (`cinatra/oas.json` → `data_flow_connections`)
6. Caller receives `findings` string

**State Management:**
- No state. The agent is stateless and ephemeral — all context arrives via inputs; all output is returned as the single `findings` string.

## Key Abstractions

**ReviewFinding:**
- Purpose: Normalized finding record emitted by the LLM
- Schema: `{ code, severity ("warning"|"suggestion"), message, location?, source: "agent-code-reviewer" }`
- Pattern: Defined in skill SKILL.md; never `"blocker"` (deterministic lint owns blockers)

**OAS Flow (`cinatra/oas.json`):**
- Purpose: Declarative agent definition consumed by the WayFlow runtime
- Pattern: Three nodes (Start → ApiNode → End), two ControlFlowEdges, five DataFlowEdges; `$referenced_components` map holds all node definitions

**Skill (`skills/code-review-methodology/SKILL.md`):**
- Purpose: Structured LLM instruction document matched by `agent_id: "@cinatra-ai/code-reviewer-agent"`
- Pattern: `match_when.agent_id` field causes the platform to inject the skill into the LLM call automatically

## Entry Points

**WayFlow runtime invocation:**
- Location: `cinatra/oas.json` → `start_node.$component_ref: "start"`
- Triggers: Orchestrator or `/chat` authoring pipeline calling the agent with an `oasJson` payload
- Responsibilities: Validate inputs (only `oasJson` is required), fan out data to the ApiNode

**CI gate (standalone):**
- Location: `extension-kind-gate.mjs` → `main()`
- Triggers: `node extension-kind-gate.mjs --package-root .` in `.github/workflows/ci.yml` → `kind-gates` job
- Responsibilities: Read `package.json`, detect `cinatra.kind: "agent"`, parse `cinatra/oas.json`, scan for retired CRM primitives in `system`/`user`/`description` fields

## Architectural Constraints

- **Threading:** Single-threaded Node.js event loop (gate script); no workers
- **Global state:** No module-level mutable state; `BANNED_PRIMITIVES`, `LLM_VISIBLE_FIELDS`, `PRIMITIVE_PATTERNS` are module-level constants in `extension-kind-gate.mjs`
- **Circular imports:** None — only one source file (`extension-kind-gate.mjs`)
- **No registry dependency:** `extension-kind-gate.mjs` uses only Node builtins; zero npm dependencies so CI passes without registry access
- **Output envelope:** LLM response MUST be `{"findings":[...]}` not a bare array — enforced by the skill and documented in the OAS system prompt

## Anti-Patterns

### Bare array output from LLM

**What happens:** LLM returns `[{...}, {...}]` instead of `{"findings":[...]}`
**Why it's wrong:** WayFlow's DataFlowEdge extracts the `findings` key from the response — a bare array causes `Cannot index array with string "findings"` at runtime
**Do this instead:** Always return `{"findings":[...]}` including when the array is empty; skill and OAS system prompt both enforce this

### Emitting "blocker" severity findings

**What happens:** Code reviewer emits a finding with `severity: "blocker"`
**Why it's wrong:** Blockers are owned by the deterministic lint; the advisory agent is limited to `"warning"` and `"suggestion"` only
**Do this instead:** Use `"warning"` for clear hygiene violations; use `"suggestion"` for style and non-critical gaps

## Error Handling

**Strategy:** Graceful degradation with a wrapped findings envelope

**Patterns:**
- OAS parse failure: return `{"findings":[{"code":"unparseable_oas","severity":"suggestion",...}]}` — never throw
- Clean OAS: return `{"findings":[]}` — never return `null`, undefined, or a bare array
- CI gate: exits 0 on pass, exits 1 with bullet-list errors on violation; catches unexpected exceptions in `main()` and exits 1

## Cross-Cutting Concerns

**Logging:** CI gate uses `console.log` (pass) and `console.error` (violations); no application-level logging in the agent OAS
**Validation:** Deterministic-first pattern — the Cinatra platform runs a structural validator before this agent; this agent adds advisory checks only
**Authentication:** Not applicable — agent identity handled by the Cinatra platform; `agent_id: "code-reviewer-agent"` in the ApiNode payload associates the run

---

*Architecture analysis: 2026-06-09*
