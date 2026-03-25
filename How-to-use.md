Now that you have built a powerful "Agentic Context Engine," the way you develop software shifts from *writing every line from scratch* to *orchestrating AI agents* using the rules you just defined.

How you actually use this depends on your tooling (e.g., an AI IDE like **Cursor** or **Windsurf**, a CLI agent like **Aider** or **Cline**, or simply uploading files to a web UI). 

Here is the step-by-step guide on how to integrate this repository structure into your daily development workflow.

---

### 1. The Tooling Integration
If you are using an AI-native IDE (like Cursor), you can reference these files directly in the chat using the `@` symbol. If you are using a CLI tool, you pass them as context flags.

The golden rule is: **Never talk to the generic AI. Always invoke a specific agent and feed it the relevant skills.**

---

### 2. The Daily Workflow: Step-by-Step

#### Phase 1: Planning a New Feature
Before writing code, you need to decide *how* it fits into the system. You will act as the Product Manager, and the AI acts as the Architect.

* **Your Goal:** Add a "Loyalty Points" system to the coffee shop.
* **The Prompt:** > "Act as the `@go-architect.md`. Read the `@DOMAIN.md`. I need to add a feature where customers earn 1 point for every $1 spent on an `Order`. Read `@go-backend-engineer/SKILL.md` and `@data-access.md`. Write an ADR for how we should store and calculate these points using the `@adr-template.md`."
* **What Happens:** The AI doesn't just guess. It knows an `Order` is a strict business term, it knows you prefer raw SQL over ORMs, and it outputs a perfectly formatted ADR for you to approve.

#### Phase 2: Implementation (Writing the Code)
Once the ADR is approved, you switch from the Architect to the Developer.

* **Your Goal:** Create the database repository for Loyalty Points.
* **The Prompt:** > "Act as a senior Go developer. Read the ADR we just created. Using the patterns in `@data-access.md` and `@clean-code/SKILL.md`, implement the `LoyaltyRepository` interface and its PostgreSQL implementation. Remember to pass `context.Context` everywhere."
* **What Happens:** The AI writes idiomatic Go code (no Java-style abstractions) and correctly uses `database/sql` or `sqlx` as dictated by your skills folder.

#### Phase 3: Writing Tests
You don't write boilerplate tests anymore; you instruct the test agent to do it based on your strict standards.

* **Your Goal:** Test the new `LoyaltyRepository`.
* **The Prompt:** > "Act as the `@test-automator.md`. Read `@testing-patterns.md`. Write a table-driven test for the `CalculatePoints` function in `loyalty_service.go`. Do not use external mocking frameworks; use manual interface mocks as defined in our skills."

#### Phase 4: Code Review (The PR Stage)
Before you commit or open a Pull Request, have your AI reviewer act as a ruthless pair programmer.

* **Your Goal:** Ensure the code won't leak goroutines or violate business rules.
* **The Prompt:** > "Act as the `@code-reviewer.md`. Read the git diff for my current changes. Review this code strictly against `@go-code-review/SKILL.md` and `@DOMAIN.md`. Output your findings using the `@pr-review-template.md`."
* **What Happens:** The AI ignores formatting (because you told it to in the skill file) and focuses on finding swallowed errors, pointer abuse, and domain violations.

---

### 3. Best Practices for this Workflow

1.  **Keep Context Windows Clean:** Do not load the *entire* `.claude` folder into every prompt. If you are writing a database query, you don't need the K8s skills. Feed the AI *only* the specific agent, the `DOMAIN.md`, and 1-2 relevant `SKILL.md` files.
2.  **Update the Skills, Not the Prompts:** If the AI makes a mistake (e.g., it writes a deep nested `if/else` block instead of an early return), **do not just fix the code**. Go into `.claude/skills/clean-code/SKILL.md` and add a rule: *"Always use early returns to avoid nesting."* Your system gets smarter over time.
3.  **The "Chain of Thought" Handover:** You can have agents talk to each other's outputs. You can tell the Go Architect to output an ADR, save that ADR, and then tell the Security Engineer: *"Read this ADR and evaluate the risks based on `@docker-security.md`."*

By strictly enforcing this structure, you stop treating the LLM like a magic autocomplete, and start treating it like a highly disciplined engineering team that follows your exact company standards.