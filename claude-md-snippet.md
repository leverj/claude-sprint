## CLAUDE.md Snippet

Add the following to your project's `CLAUDE.md` file to remind Claude about the sprint workflow in every session:

---

## Sprint Workflow

This project uses the `/sprint` skill for development workflow management.

- Requirements are tracked as GitHub Issues with structured acceptance criteria (WHEN/THEN/SHALL format)
- Architectural decisions are recorded in `.dev/decisions/`
- Run `/sprint status` to see current state before starting work
- Run `/sprint pick` to claim and implement a requirement
- Run `/sprint plan` to create new requirements
- Run `/sprint decide` to record a decision
- Run `/sprint refine` to groom rough issues into implementable ones
