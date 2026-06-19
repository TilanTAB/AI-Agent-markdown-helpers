# AI Agent markdown helpers

Opinionated markdown configuration for AI coding assistants — built for
[Claude Code](https://www.anthropic.com/claude-code), but useful for any agent
that reads global or project-level instruction files.

Two files you can drop into your own setup:

| File | What it is | Where it goes |
|------|------------|---------------|
| [`CLAUDE.md`](CLAUDE.md) | Global directives that turn the assistant into a blunt senior-architect reviewer: anti-hallucination rules with `[Verified]`/`[Inferred]`/`[Uncertain]` labels, mandatory stress-testing of every proposal, plan-before-build, a teaching mode, and a "top 2 risks" footer on every reply. | `~/.claude/CLAUDE.md` (global) or a project root |
| [`skills/review/SKILL.md`](skills/review/SKILL.md) | A language-agnostic code-review skill: 12 critical review areas (async/concurrency, thread safety, performance, OWASP security, resource management, error handling, scalability, SOLID, testability, API design, design maintainability, contract verification), severity labels, and stack-specific checklists for Python, JS/TS, C#/.NET, SQL, and IaC. | `~/.claude/skills/review/SKILL.md` |

## How to use

### Global directives (`CLAUDE.md`)

Copy it to your user-level config so it applies to every project:

- **macOS / Linux:** `~/.claude/CLAUDE.md`
- **Windows:** `%USERPROFILE%\.claude\CLAUDE.md`

Or drop it in a project root as `CLAUDE.md` to scope it to a single repo.

### Review skill (`skills/review/SKILL.md`)

Copy the `skills/review/` folder into your `~/.claude/skills/` directory, then
invoke it inside Claude Code:

```text
/review                  # review staged/unstaged changes
/review src/MyFile.py    # review a specific file
/review src/             # review a directory
/review --pr 123         # review a pull request
```

## Make it yours

These reflect one engineer's taste — direct tone, heavy on production-failure
paranoia, allergic to confidently-wrong answers. Fork it, dial the snark up or
down, swap the stack checklists for your own. That's the point.

## License

[MIT](LICENSE) © TilanTAB
