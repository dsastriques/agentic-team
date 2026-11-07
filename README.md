# Agentic team

Skills that form a multidisciplinary software team. Each expert is a skill; they collaborate to turn a product vision into working software.

## Skills

| Skill | Role | Invoke |
|---|---|---|
| [coordinator](skills/coordinator/SKILL.md) | Leads research and functional requirements, keeps the project log, hands technical work to the architect | `/coordinator [task]`, `/coordinator -r [task]` for research only |
| [architect](skills/architect/SKILL.md) | Technology selection, system design, deployment and test strategy; plans, delegates and reviews implementation | `architect design`, `architect impl <task>` |
| [designer](skills/designer/SKILL.md) | Usability, accessibility (WCAG 2.2 AA), design patterns, design system and components, micro-animations (named motion catalog), responsive screens and full flows built in Figma from the requirements, with implementation metadata for a frontend skill; reviews built UI | `designer system [scope]`, `designer design [feature]`, `designer review <target>` |
| [frontend](skills/frontend/SKILL.md) | React / React Native (Expo) and TanStack Query; bundling, lazy loading, micro frontends, accessibility, localization, REST consumption; builds screens, components and motion presets from the designer's handoff contract and the architect's API contracts; reviews FE code | `frontend setup [scope]`, `frontend build <SCR-id \| feature \| task>`, `frontend review <target>` |
| [backend](skills/backend/SKILL.md) | Python and Java services, Postgres, Redis, SOLID and GoF patterns, REST / API design, AI agent implementation; implements backend tasks from the architect's design (hexagonal, DDD), answers the frontend's API requests, hands tests to qa and runtime needs to devops; reviews backend code | `backend setup [scope]`, `backend build <task \| plan>`, `backend api [REQ-API-xx]`, `backend review <target>` |

Planned: `devops`, `qa` (called by the architect).

## Flow

1. **Coordinator** researches, writes requirements (`BR-xx`, `FR-xx`, `NFR-xx`) and logs technical topics without pausing.
2. When the investigation ends, the coordinator hands a brief to the **architect**, unless the run is `-r`.
3. The **designer** builds the design system, components, responsive screens and flows in Figma from the requirements, with UX acceptance criteria (`UX-xx`) and per-screen implementation metadata (`docs/design/`).
4. The **architect** designs (`docs/architecture/`), then implements tasks through the backend, frontend, devops and qa skills and opens PRs.
   Backend tasks go to the **backend** skill, which builds inside-out (domain → use cases → adapters) against the contracts, answers the frontend's `REQ-API-xx` requests with proposed contract changes, and records runtime needs for devops in `docs/backend/ops-handoff.md`.
   UI tasks go to the **frontend** skill, which builds screens from the designer's screen JSON and the API contracts, logging `DESIGN-GAP`, `TBD-ARCH` and `TBD-BE` items instead of inventing.
   The designer reviews built UI for usability, accessibility and design-system compliance.
5. The owner is asked only for decisions that are theirs.

## Layout

```
skills/<skill-name>/SKILL.md   # plus any references/ or scripts/ the skill needs
```

## Installing

Upload a skill folder (zipped) in claude.ai under Settings → Capabilities → Skills, or copy it into `~/.claude/skills/` for Claude Code.
