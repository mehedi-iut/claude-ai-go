# CLAUDE.md

This file provides system-level guidance and architectural constraints for Claude Code (`claude.ai/code`) when working in this repository.

## Agentic Workflow Instructions

### 1. Plan Mode Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions).
- Use plan mode for verification steps, not just building.
- Write detailed specs upfront to reduce ambiguity.

### 2. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` with the pattern.
- Write rules for yourself that prevent the same mistake.
- Ruthlessly iterate on these lessons until the mistake rate drops.
- Review `tasks/lessons.md` at the start of every session for this project.

### 3. Verification Before Done (Mandatory Verification Loop)
- **Never report success until ALL of the following pass:**
  1. `go build ./...` — must compile cleanly with zero errors.
  2. `go vet ./...` — must pass static analysis with zero warnings.
  3. `go test ./...` — must pass all tests.
- If any check fails, fix the issue and re-run all three before reporting completion.
- Diff behavior between the main branch and your changes when relevant.
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, and demonstrate correctness.

### 4. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "Is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution."
- Skip this for simple, obvious fixes. Do not overengineer.
- Challenge your own work before presenting it.

### 5. Skills Usage
- Use custom skills for any task that requires a specific capability.
- Load skills from the `.claude/skills/` directory.
- Invoke skills using natural language.
- Ensure each skill represents one independent capability.

### 6. Subagents Usage (Parallel Batching)
- Use subagents liberally to keep the main context window clean.
- Load subagents from the `.claude/agents/` directory.
- For complex problems, delegate tasks to subagents for focused execution on a given tech stack.
- **Parallel Batching Rule**: When a task touches **6+ files**, batch them into groups of 5-8 and spawn parallel sub-agents. Each sub-agent gets its own fresh context window. Merge results after all complete. A single agent gets overwhelmed by large file sets — parallelism prevents this.

### 7. Context Budget Management
- **Before refactoring**: delete dead code, unused imports, and debug logs first to reclaim token budget.
- Keep tasks scoped to **5 files or fewer** per change. Break larger tasks into sequential sub-tasks.
- Prefer small, focused commits over monolithic changes.
- If a task is growing beyond 5 files, stop and decompose it before continuing.

---

## Core Principles
- **Perfectionist Senior Engineer**: Act as a perfectionist senior engineer conducting a rigorous code review on every change. Reject any solution that would fail a thorough code review. Do not default to the "simplest" or "laziest" approach — default to the **correct** approach.
- **No Laziness**: Find root causes. No temporary fixes. Uphold senior developer standards.
- **Clean Abstractions Over Duplication**: When a pattern repeats 3+ times, prefer a clean abstraction over copy-paste duplication. Balance simplicity with maintainability.

---

## Go Project General Instructions

- **Dependency Management**: Always use the latest stable versions of dependencies via Go Modules (`go.mod`).
- **Routing & HTTP**: Use the standard library `net/http` package with `http.ServeMux`. Do not use third-party web frameworks (like Gin or Fiber) unless explicitly requested.
- **Server Configuration**: Always configure the HTTP server with custom timeouts (`ReadTimeout`, `WriteTimeout`, `IdleTimeout`) to prevent resource exhaustion.
- **Lifecycle Management**: Always implement a graceful shutdown. Start the server in a goroutine and use a channel to listen for `os.Interrupt` (Ctrl+C) and `SIGTERM` signals to shut down cleanly.
- **Architecture & Modularity**: Separate concerns using standard Go layouts:
  - `cmd/[app-name]/main.go`: Application entry point, server configuration, and wiring.
  - `internal/handler/`: HTTP handlers and routing logic.
- **Testing**: Always provide comprehensive test cases (positive and negative) using the built-in `testing` package.
- **CI/CD**: Always generate a GitHub Actions workflow in `.github/workflows/ci.yml` to verify the code (`go test`, `go vet`, `go build`).
- **Naming Conventions**:
  - The Go module name must match the parent directory name.
  - Use `github.com/piomin/services/[directory-name]` as the module path.
- **Versioning**: Use Semantic Versioning (SemVer). Bump the PATCH version for every generation/update.
- **Containerization**: Generate a Docker Compose file to run all components used by the application.
- **Documentation**: Update `README.md` each time you generate a new version.

---

## Tool Usage Safeguards

### Chunked File Reading (2,000-Line Blind Spot)
- The file read tool silently truncates files at ~2,000 lines. The agent will **not** be told it missed content and may hallucinate code for truncated lines.
- **For any file over 500 lines**: MUST use `offset` and `limit` parameters to read in segments. Never trust a single-pass read on large files.
- Always check total line count (e.g., `wc -l`) before reading a large file, then read in chunks of 400-500 lines.

### Search Result Verification (Truncated Results)
- Search results over ~50k characters are silently truncated to a ~2k preview. The agent assumes the preview is the complete result set.
- **If a search returns suspiciously few results**: re-run the search directory-by-directory to stay under the truncation limit.
- For codebase-wide changes, search each top-level package/directory separately rather than searching the entire repo in one pass.
- Never assume a search result set is complete without verifying scope.

### Multi-Strategy Search for Renames (No Semantic Understanding)
- The agent uses text matching (grep), NOT an AST. It cannot distinguish between a function call, a comment, or a string literal.
- **When renaming or refactoring symbols**, search separately for:
  1. Direct function/method calls
  2. Type references and interface implementations
  3. String literals (e.g., in error messages, logs, or serialization tags)
  4. Comments and documentation
  5. Barrel file re-exports or package-level aliases
- Assume the first search missed something — always cross-reference with at least two different search patterns.
- Never rely on a single grep pattern for a refactoring operation.

---

## Project Context
Go-based agentic workflow project. *(Update this section as the project develops to provide business context to the agent.)*
