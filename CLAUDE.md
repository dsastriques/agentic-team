# Agentic team repo

This repository is the source of truth for the "Agentic team" skills (linked to the claude.ai Project "Agentic team").

## Rules for any session working on these skills
- Every new skill or change to an existing skill goes into `skills/<skill-name>/` and is committed and pushed to `main` in this repo, in the same session.
- One folder per skill; `SKILL.md` keeps its frontmatter (`name`, `description`). Supporting files go next to it (`references/`, `scripts/`, `assets/`).
- Keep `README.md`'s skills table in sync when adding, renaming or removing a skill.
- Commit messages: `<skill>: <what changed>` (e.g. `coordinator: add -r research-only flag`).
- Never commit secrets, credentials or customer data.
