# PR Review Agent
Perform a structured code review for pull request $ARGUMENTS.
## Arguments
`$ARGUMENTS` is the PR number/identifier plus optional flags, e.g. `123 --comment --output-dir=docs/reviews`.
- **PR identifier** — first non-flag token. Passed to all `gh pr` commands below.
- **`--comment`** — post the "Code Review Summary" block as a PR comment via `gh pr comment`. **Default: off.** Omit unless this flag is present.
- **`--output-dir=<path>`** — directory to write the review doc into. **Default: `.claude/.local-docs/pr-reviews`.**
## Setup
First, gather all context needed for the review:
1. Fetch PR metadata (title, description, base branch):
   ```
   gh pr view $ARGUMENTS --json number,title,body,baseRefName,headRefName
   ```
2. Fetch the full diff:
   ```
   gh pr diff $ARGUMENTS
   ```
3. Fetch the file list (for additional context if needed):
   ```
   gh pr view $ARGUMENTS --json files
   ```
4. Fetch CI status:
   ```
   gh pr checks $ARGUMENTS
   ```
5. Fetch existing review activity so you don't re-report what's already been raised (Copilot, teammates, prior review runs):
   ```
   gh pr view $ARGUMENTS --json reviews,comments
   gh api repos/{owner}/{repo}/pulls/<number>/comments
   ```
6. If the PR title or head branch contains an OVP ticket number and Atlassian tools are available, fetch the Jira issue for stated intent and acceptance criteria. Skip silently if unavailable.
Derive the output path from the PR's **`headRefName`** (not the locally checked-out branch — you may be reviewing someone else's PR), with `/` replaced by `-`:
`<output-dir>/<slugified-head-ref>-pr<number>.md`, where `<output-dir>` is the `--output-dir` value (default `.claude/.local-docs/pr-reviews`).
Create the output directory if it does not exist:
```
mkdir -p <output-dir>
```
---
## Review Instructions
Analyze the diff and surrounding context. Before flagging a CODE_QUALITY duplication finding, actively search the broader codebase (not just the diff) for existing utilities, services, hooks, or components that already solve the same problem, and check whether the standard library, a platform feature, or an already-installed dependency already covers it (date math, deep-equality, debounce, retry-with-backoff, string formatting, and similar are common candidates for hand-rolled reimplementation) — grep for comparable function signatures, business logic, or UI components. Point to the specific existing file/symbol or built-in that could be reused rather than noting duplication in the abstract.
Identify issues across all of the following categories:
- **CORRECTNESS** — Logic errors, off-by-one, incorrect conditionals, missing edge cases (null, empty, boundary values), race conditions, incorrect assumptions about data state, partial-update handlers that silently no-op on clear (e.g. `field?.let { x = it }` patterns that can't distinguish "not provided" from "explicitly cleared to null/blank" — check whether clearing a field is a supported user flow and whether the handler actually supports it), a shorter implementation chosen over an equally simple alternative that handles edge cases correctly — when two approaches are the same size, the edge-case-correct one should win
- **SECURITY** — Secrets/credentials/API keys in code or logs, injection vulnerabilities (SQL, command, path traversal), broken auth or missing authorization checks, PII/PHI exposure in logs or error responses, insecure deserialization, unsafe input handling, client-supplied audit/identity fields (e.g. `updatedBy`, `createdBy`) that should instead be derived from the authenticated principal (JWT `sub` claim, session user, etc.) rather than trusted from the request body
- **PERFORMANCE** — N+1 query patterns, unbounded DB calls, missing pagination, blocking I/O in coroutine/async contexts, unnecessary repeated computation, over-fetching, a SET/PATCH (or similar mutation) endpoint backing a UI flow that supports multi-select without accepting a batch of resource identifiers — forcing the frontend to issue one sequential call per selected item instead of a single bulk call; check whether the endpoint contract should accept a list of IDs/resources
- **ARCHITECTURE** — Layer violations (business logic in controllers or agents), LLM used where deterministic code should be used, agent responsibilities that belong in a service, coupling that harms testability or extensibility, missed or premature abstractions — including interfaces, generic wrappers, or configuration options added beyond what the current change actually requires (YAGNI; ask "do we need this, or does the simpler existing path already cover it?"), a new third-party dependency introduced where the standard library, an already-installed dependency, or an existing in-repo utility already solves the problem
- **CODE_QUALITY** — DRY violations, functions doing too many things, poor naming, magic constants, dead code, commented-out blocks, TODOs without ticket references, excessive nesting or cyclomatic complexity, hand-rolled inline type shapes (e.g. an anonymous object type for a function parameter) duplicating an existing named type/interface — prefer reusing the existing type, new logic/components/utilities introduced when an equivalent already exists elsewhere in the codebase — cite the existing implementation and recommend reuse or extraction over reimplementation, boilerplate or scaffolding not required by the task scope, a diff that touches more files or adds more indirection than the stated change needs when a smaller, more direct edit (or a deletion) would do, a deliberate simplification or known limitation (naive O(n²) scan, global lock, hardcoded heuristic) left without a comment naming the ceiling and the upgrade path (the `ponytail:` convention in CLAUDE.md, where adopted), hand-rolled logic that reinvents something the standard library, the runtime/platform, or an already-installed dependency already provides, dead flexibility — parameters, options, branches, or feature flags that no current caller ever exercises, unused exports
- **ERROR_HANDLING** — Missing try/catch in I/O paths, generic catch blocks masking real errors, swallowed exceptions, no retry logic on transient failures, poor error messages, missing circuit breakers on external calls, frontend write operations (create/update/delete calls) with no failure path — a rejected promise that never surfaces a toast or inline error to the user
- **TESTABILITY** — Untestable design (static calls, hard-coded dependencies), missing tests for introduced logic, happy-path-only tests, flaky test patterns, new validation rules or auth-derived fields (e.g. required/optional constraints, `updatedBy` matching the authenticated subject) introduced without tests covering the pass/fail scenarios
- **OBSERVABILITY** — Missing logging at key decision points, PII in logs (also flag under SECURITY), no tracing/span context propagated, metrics not emitted for new operations
- **API_CONTRACT** — Breaking changes to public interfaces without versioning, response shape changes that could break consumers, missing or changed input validation, backward-incompatible schema changes without migration, Bean Validation constraints declared on a nested DTO that won't actually be enforced because the parent reference is missing `@field:Valid`/`@Valid` — check the full reference chain from the validated resource entrypoint down, not just the class where the constraint is declared
- **LANGUAGE_QUALITY** — Non-idiomatic Kotlin (Java-isms, `!!` overuse, `.let`/`.also` misuse), nullable anti-patterns, coroutine scope misuse, missed use of data classes, sealed classes, enums for fixed string/value sets, or `when` expressions. Non-idiomatic TypeScript/React: incorrect hook dependency arrays, client components that should be server components (or vice versa), hand-typed API responses instead of generated types from `_generated/`, new SWR usage (legacy — Tanstack Query is preferred), missing `await` on async calls, `any` where a specific type exists, clearable inputs (`clearable`, `allowClear`, etc.) whose `onChange`/handler has no branch for the null/cleared value, multiple distinct interactive actions grouped under a single `Menu.Target` (or similar single semantic target) such that they read as one control to assistive tech
---
## Rating Each Issue
Assign one of the following ratings:
- **CRITICAL** — Must fix before merge. Security vulnerability, data loss risk, or correctness bug that will cause production failures.
- **HIGH** — Should fix before merge. Significant risk, performance regression, or architectural violation.
- **MEDIUM** — Should fix soon. Code quality, maintainability, or minor logic concern.
- **LOW** — Nice to fix. Style, idiom, or minor improvement.
Assign one of the following origins:
- **INTRODUCED** — This issue is new in this PR's diff.
- **PRE-EXISTING** — This issue exists in context lines or surrounding code not touched by this PR.
If a pre-existing issue is directly worsened or relied upon by introduced code, note that explicitly.
---
## Validation Pass (required before writing output)
After identifying all candidate issues, validate each **CRITICAL** and **HIGH** issue before including it in the output. For each one:
1. **Verify reachability** — Read the actual call sites for the affected function/method. Is the problematic code path reachable in production? Is it user-facing, a dev/admin tool, or dead code?
2. **Check preconditions** — Are there constraints (DB unique constraints, auth guards, UI flow restrictions) that prevent the issue from occurring in practice?
3. **Assess actual impact** — Given reachability and preconditions, is the severity correct? Downgrade if the path is rarely or never triggered by users.
If a CRITICAL or HIGH issue cannot be verified as reachable and impactful after this investigation, downgrade it to MEDIUM or LOW and note the reason (e.g., "only reachable via admin endpoint", "unique constraint prevents this state").
Do not skip this step. A finding that looks severe in isolation may be low-risk in context.
---
## Output: Full Review Document
Write the following to `.claude/.local-docs/pr-reviews/<branch-name>-pr<number>.md`:
```markdown
# PR Review: <PR Title>
**PR:** #<number> | **Branch:** `<head-ref>` | **Base:** `<base-branch>`
**Reviewed:** <date> | **CI:** <passing / failing (which checks) / pending>
---
## Issues
<!-- One block per issue, sorted CRITICAL → HIGH → MEDIUM → LOW, numbered continuously across all bands -->
### Issue N — [CRITICAL|HIGH|MEDIUM|LOW] CATEGORY — Short Title
**File:** `path/to/file.kt` (line N)
**Origin:** INTRODUCED | PRE-EXISTING
**Description:**
Clear explanation of why this is a problem and what could go wrong.
**Suggested Fix:**
Concrete recommendation. Include a short code snippet if it aids clarity.
---
## PR Summary Comment
> Copy and paste the block below into the GitHub PR comment field.
---
## Code Review Summary
**CRITICAL**
- **#N — TITLE OF ISSUE**
  One-sentence description of the problem. Suggested fix in brief.
**HIGH**
- **#N — TITLE OF ISSUE**
  One-sentence description. Suggested fix in brief.
**MEDIUM**
- **#N — TITLE OF ISSUE**
  One-sentence description. Suggested fix in brief.
**LOW**
- **#N — TITLE OF ISSUE**
  One-sentence description. Suggested fix in brief.
```
Do not add any generated-by/agent attribution footer inside the paste block — GitHub content must read as human-authored (see `.claude/CLAUDE.md`).
Issue numbers are continuous across all severity bands (e.g., if there are 2 CRITICAL issues, the first HIGH issue is #3).
Omit any rating band from the summary that has no issues.
---
## Posting the Comment (only if `--comment` was passed)
Skip this section entirely if `--comment` was not in `$ARGUMENTS`.
Post the "Code Review Summary" block (from "**CRITICAL**" through the last rating band, exactly as written to the output doc) to the PR:
```
gh pr comment <number> --body-file <path-to-output-doc-summary-section>
```
---
## Constraints
- Only flag real issues. Do not pad the review with nitpicks that don't add value.
- Cross-check against existing review comments fetched in Setup. If an issue was already raised by a human or Copilot, don't re-report it as a new finding — include it only if still unresolved and material, marked "(previously flagged by <who>)".
- Be specific about file and line number. Vague findings are not useful.
- Do not hallucinate code that isn't in the diff. If context is ambiguous, say so explicitly.
- Distinguish clearly between INTRODUCED and PRE-EXISTING so the author knows what they own.
- For ARCHITECTURE issues, name the specific design principle being violated (e.g. "business logic in agent layer", "LLM used for deterministic calculation").
- For SECURITY issues involving PII/PHI, also flag under OBSERVABILITY if the exposure is via logging.
- Sort issues within each rating band by file path for easier navigation.
- Suggested fixes should be the smallest correct change that addresses the root cause — do not recommend broad rewrites, speculative abstractions, or new dependencies the PR doesn't need.
- When flagging complexity findings (premature abstraction, unneeded dependency, reinvented stdlib, dead flexibility, oversized diff), never flag input validation at trust boundaries, error handling that prevents data loss, security/auth checks, accessibility, tests, or anything explicitly requested in the PR description or linked ticket — and don't re-report a deliberate simplification that already carries a `ponytail:` comment naming its ceiling and upgrade path, since that's already documented, not an oversight.

