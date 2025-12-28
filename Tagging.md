For LLM-powered bots like Claude Code, the label style matters less than you might think, since the bot is generating and parsing labels—not humans typing them. That said, here's my take:

**`Type: Bug` style is slightly better for LLM workflows** for a few reasons:

1. **Natural language alignment** — LLMs are trained heavily on natural language, so "Type: Bug" reads more like how they'd naturally express categorization. Less cognitive mismatch between the label and how the model reasons about it.

2. **Self-documenting in prompts** — When you feed issue metadata to Claude, `Type: Bug, Priority: High, Area: Backend` reads more naturally in a prompt than `kind/bug priority/high area/backend`.

3. **Easier to generate correctly** — LLMs are less likely to hallucinate malformed labels when the format mirrors natural English ("Status: Needs Triage" vs remembering whether it's `status/needs-triage` or `status/needstriage`).

**Practical label schema for bot-driven workflows:**

| Category | Labels | Bot trigger purpose |
|----------|--------|---------------------|
| **Action** | `Action: Autofix`, `Action: Review`, `Action: Generate Tests` | Direct commands for the bot |
| **Status** | `Status: Bot Working`, `Status: Awaiting Review`, `Status: Bot Failed` | Track automation state |
| **Type** | `Type: Bug`, `Type: Feature`, `Type: Refactor` | Context for how bot should approach |
| **Priority** | `Priority: Critical`, `Priority: Low` | Helps bot prioritize queue |
| **Scope** | `Scope: Small`, `Scope: Large` | Hints about expected complexity |

**Key insight:** Add an explicit `Action:` category for bot triggers. Rather than overloading `Type: Bug` to mean "fix this," use `Action: Autofix` so intent is unambiguous. The bot sees the label, knows exactly what's expected, and you can combine it with type/area labels for context.

---

*Thinking...*

> **Evaluating User Intent**
> 
> I'm currently dissecting the user's prompt, specifically keying in on their reference to a prior exchange. The goal is to accurately grasp the nuances of their request, noting the context of the structured tags. The existing prompt sets the stage; I need to understand the intent.
> 
> 
> **Deciphering User Goal**
> 
> My assessment now involves mapping the user's articulated need onto actionable steps. I'm focusing on the AI agent interaction, and the use of the structured tags. The goal is to facilitate a smooth handoff between user intent and agent execution. I'm evaluating the optimal format for the AI agent trigger.
> 
> 
Based on the context provided and your goal of triggering AI agents (like Claude Code) to perform work, here is a breakdown of the best tagging strategy.

### Is `Type: Bug` better for LLMs?

**No.** Even though LLMs are generating the code or handling the issue, **structured, lowercase, slash-delimited tags (e.g., `action/generate-test`) remain superior** to "natural language" tags (e.g., `Type: Bug`) for triggering bots.

Here is the reasoning:

1.  **The Trigger Mechanism is Deterministic:** While the *worker* is an LLM (Claude), the *trigger* is usually a deterministic script (GitHub Actions, Webhooks, or a Probot). These scripts look for exact string matches. It is much safer to write a workflow that listens for `bot/fix` than one that tries to parse `Please Fix This` or handle the casing variations of `Type: Fix`.
2.  **Token Efficiency & Context:** When you pass issue context to an LLM, concise, structured tags act as clear instructions. `scope/frontend` is a clearer directive to an AI than a loose category.
3.  **Avoid Hallucination:** If you ask an LLM to "apply the right label" based on a natural language schema, it might hallucinate `Type: Critical Bug` when your system only recognizes `Type: Bug`. Structured schemas (`priority/p0`) have stricter boundaries that you can enforce in the prompt given to the AI.

### Recommended Tagging Schema for AI Bots

If your goal is to have an AI agent pick up a ticket and do actual coding work, you should use a schema that defines the **Action** required and the **Scope** of the work.

#### 1. The "Action" Prefix (What should the bot do?)
Instead of just describing the issue (`kind/bug`), use labels that act as commands for the bot.

*   `bot/fix`: Tells the agent to attempt to fix the bug described.
*   `bot/generate-tests`: Tells the agent to write unit tests for the code in the issue/PR.
*   `bot/review`: Triggers an AI code review.
*   `bot/document`: Tells the agent to generate or update documentation based on the code changes.
*   `bot/explain`: Asks the agent to post a comment explaining what the code does (useful for complex legacy code).

#### 2. The "Scope" Prefix (Where should the bot look?)
LLMs work best when focused. Global context can be expensive and confusing.

*   `scope/frontend`: Focus on the UI directory.
*   `scope/api`: Focus on backend endpoints.
*   `file/database`: Focus specifically on schema or migration files.

#### 3. The "State" Prefix (What is the bot doing now?)
Since AI takes time to "think" and generate code, you need status labels so humans don't interfere while the bot is working.

*   `status/ai-processing`: The bot has acknowledged the command and is generating a response.
*   `status/human-review-needed`: The bot has finished and needs a human to verify the code.

### Example Workflow

If you want Claude Code to fix a bug in your React app, you would apply these labels:

1.  **Human applies:** `bot/fix`, `scope/frontend`, `priority/high`
2.  **Automation script:** Detects `bot/fix`.
3.  **Script:** Adds `status/ai-processing`.
4.  **AI Agent:** Reads the issue, sees `scope/frontend`, ingests the relevant files, generates a Pull Request.
5.  **Script:** Removes `status/ai-processing`, adds `status/human-review-needed`.

### Summary
Stick to the **`prefix/value`** convention (lowercase, no spaces). It is easier to program the "glue code" that connects your repository to the LLM API, and it provides clear, unambiguous instructions to the model regarding its specific task.
