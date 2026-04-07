---
codd:
  node_id: req:codd-v2
  type: requirement
  depends_on:
  - id: req:system
    relation: supersedes
    semantic: governance
  confidence: 0.95
  source: human
---

# CoDD Requirements v2.0 — Self-Improving Design-Driven Development

## 1. Overview

CoDD (Coherence-Driven Development) is a CLI tool that maintains bidirectional
traceability between requirements, design documents, implementation, and tests.
It operates in greenfield (spec-first) and brownfield (extract-first) modes.

**Core thesis**: When an AI has access to design documents describing intended
behavior, it fixes bugs more accurately than without — because it can distinguish
"implementation is wrong" from "test is wrong".

**Self-improvement thesis**: CoDD can improve itself by running its own pipeline
(extract → requirements → fix) against benchmark failures, creating a virtuous
cycle where each fix makes the tool better at fixing.

## 2. Design Principles (Non-Negotiable)

### P1. Design-First Consistency Chain

```
Requirements → Design Docs → Tests → Implementation
```

When a test fails, the fix direction is ALWAYS: fix implementation to match
design, not loosen tests to match implementation. Tests are derived from specs;
specs are the source of truth.

### P2. Anti-Overfitting

All improvements to CoDD MUST be generic. Specifically:

- **No benchmark-specific logic**: No conditionals, heuristics, or prompts that
  reference specific benchmark scenarios, repositories, or test case IDs.
- **No tool-specific logic**: No ESLint-specific, Jest-specific, or
  framework-specific code. Use abstract categories: "linter", "test runner",
  "build tool", "config file".
- **Abstraction test**: Before merging any fix, ask: "Would this help a project
  I've never seen?" If no, the fix is overfitting.

### P3. Source-In / Fixed-Source-Out

The fix pipeline is stateless and text-based:

1. Input: error log + current source + design docs
2. AI processing: `--print` mode (no interactive sessions)
3. Output: complete fixed source in tagged code blocks
4. Application: parser extracts blocks, writes files back

No interactive AI sessions. No agent mode for fix. The AI is a pure function:
`(error, source, spec) → fixed_source`.

### P4. Model Agnosticism

All AI invocations go through a configurable `ai_command` string. CoDD works
with any CLI tool that accepts stdin prompt and returns stdout text:

- `claude --print`
- `codex exec --full-auto`
- `ollama run llama3`
- Any future model CLI

## 3. Functional Requirements

### 3.1 Extract (Brownfield Bootstrap)

- **FR-EXT-1**: `codd extract` SHALL reverse-engineer design documents from
  existing codebases using static analysis (default) or AI-assisted extraction
  (`--ai` flag).
- **FR-EXT-2**: Extracted documents SHALL include CoDD frontmatter (node_id,
  type, confidence, dependencies) so they integrate into the dependency graph.
- **FR-EXT-3**: Static extraction SHALL produce: module map, dependency graph,
  architectural layers, interface contracts, environment variables, and build
  metadata.

### 3.2 Generate (Greenfield Document Generation)

- **FR-GEN-1**: `codd generate` SHALL produce design documents from requirements
  using the wave-based dependency order defined in `codd.yaml`.
- **FR-GEN-2**: Generated documents SHALL include meta-prompts for downstream
  artifacts (E2E tests, CI/CD pipelines).
- **FR-GEN-3**: CI/CD meta-prompts SHALL include:
  - Prerequisite validation (verify tools exist in project dependencies)
  - Runtime compatibility (detect project tool versions, generate compatible configs)
  - E2E server startup (detect web app projects, include build→start→health-check)
- **FR-GEN-4**: E2E test meta-prompts SHALL include runtime environment rules
  specifying how to start the application under test.

### 3.3 Fix (Design-Guided Auto-Repair)

- **FR-FIX-1**: `codd fix` SHALL detect failures from three sources:
  explicit files (`--ci-log`, `--test-results`), CI (`gh run view`), or
  local test execution.
- **FR-FIX-2**: Failure categories SHALL be: test, build, lint, typecheck,
  config. Detection SHALL be based on log content patterns, not tool names.
- **FR-FIX-3**: The fix prompt SHALL include:
  - Error log (categorized failure)
  - Current source code of relevant implementation files
  - Design documents mapped via the dependency graph
  - Instructions enforcing P1 (fix implementation, not tests)
- **FR-FIX-4**: The AI response SHALL contain complete fixed source in fenced
  code blocks tagged with file paths: `` ```language path/to/file ``
- **FR-FIX-5**: The fixer SHALL parse code blocks from AI output, validate that
  target paths are within the project root, and write files back to disk.
- **FR-FIX-6**: After writing fixes, the fixer SHALL re-run tests to verify.
  If tests pass, the fix is confirmed. If not, iterate up to `max_attempts`.
- **FR-FIX-7**: When local tests cannot be executed (missing server, database,
  etc.), the result SHALL be "unverified", not "fixed". False positives from
  unrunnable tests are a defect.
- **FR-FIX-8**: The fixer SHALL map test file names to implementation files
  using: (a) the dependency graph, (b) naming conventions (e.g.,
  `tests/e2e/enrollments.spec.ts` → `src/app/api/enrollments/route.ts`),
  (c) error log file paths.
- **FR-FIX-9**: The fix prompt SHALL instruct the AI to:
  - Add missing implementations described in design docs when tests expect
    features that don't exist in code
  - Follow the target framework's lint rules and naming conventions
  - Avoid reserved/global variable names
  - Create missing config files when CI tools prompt interactively
- **FR-FIX-10**: The `ai_commands.fix` configuration SHALL default to a system
  prompt optimized for code repair, not document generation.

### 3.4 Propagate & Implement

- **FR-PROP-1**: `codd propagate` SHALL generate downstream artifacts (tests,
  CI configs) from design documents using the meta-prompts embedded in them.
- **FR-IMPL-1**: `codd implement` SHALL generate implementation code from
  design documents, respecting the dependency order and module boundaries.

### 3.5 Validate & Measure

- **FR-VAL-1**: `codd validate` SHALL check document consistency: node ID
  format, dangling references, circular dependencies, wave ordering.
- **FR-MEAS-1**: `codd measure` SHALL compute coverage metrics: how many
  design nodes have corresponding implementation, tests, and CI coverage.

### 3.6 Self-Improvement Loop

- **FR-SELF-1**: CoDD SHALL be able to run its own pipeline on itself:
  `codd extract` on codd-dev → design docs → `codd fix` with benchmark
  failures as input.
- **FR-SELF-2**: Benchmark targets for self-improvement:
  - **spec-drift-bench**: All scenarios SHALL pass (qualitative gate)
  - **SWE-bench Lite**: Track resolve rate n/300 (quantitative metric)
  - **osato-lms**: Real-world brownfield validation project
- **FR-SELF-3**: Self-improvement fixes SHALL pass the anti-overfitting test
  (P2) before being committed.

## 4. Non-Functional Requirements

### 4.1 Performance

- **NFR-PERF-1**: `codd extract` (static mode) SHALL complete in <30 seconds
  for codebases up to 50,000 lines.
- **NFR-PERF-2**: `codd fix` SHALL complete one attempt in <5 minutes
  (excluding AI response time).

### 4.2 Portability

- **NFR-PORT-1**: CoDD SHALL work on Linux, macOS, and WSL2.
- **NFR-PORT-2**: CoDD SHALL support Python, TypeScript, JavaScript, and Go
  codebases. Java is symbol-extraction only.

### 4.3 Testability

- **NFR-TEST-1**: All public functions SHALL have unit tests.
- **NFR-TEST-2**: The fix pipeline SHALL be testable with mock AI responses
  (no real API calls in unit tests).

## 5. Quality Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Unit test count | >400 | 429 |
| spec-drift-bench pass rate | 100% | TBD |
| SWE-bench Lite resolve rate | >15% | TBD |
| Fix false-positive rate | 0% | >0% (backlog #9) |

## 6. Known Gaps (Backlog)

| # | Category | Description | Priority |
|---|----------|-------------|----------|
| 7 | fix | System prompt for fix is "document generator" | High |
| 8 | fix | Implementation file reverse-lookup from test names | High |
| 9 | fix | False positive when local tests can't run | High |
| 10 | fix | Code block parse fallback patterns | Medium |

## 7. Distribution Strategy

- **Primary**: SWE-bench leaderboard (13.8k stars, established credibility)
- **Secondary**: spec-drift-bench (owned benchmark, CoDD-specific scenarios)
- **Validation**: osato-lms (real brownfield project, ongoing)
