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

### 3. Verification Before Done
- Never mark a task complete without proving it works.
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

### 6. Subagents Usage
- Use subagents liberally to keep the main context window clean.
- Load subagents from the `.claude/agents/` directory.
- For complex problems, delegate tasks to subagents for focused execution on a given tech stack.

---

## Core Principles
- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Uphold senior developer standards.

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

## Project Context
Go-based agentic workflow project. *(Update this section as the project develops to provide business context to the agent.)*
