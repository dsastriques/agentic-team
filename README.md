# Agentic team

Skills that form a multidisciplinary software team. Each expert is a skill; they collaborate to turn a product vision into working software.

## Skills

| Skill | Role | Invoke |
|---|---|---|
| [coordinator](skills/coordinator/SKILL.md) | Leads research and functional requirements, keeps the project log, hands technical work to the architect | `/coordinator [task]`, `/coordinator -r [task]` for research only |
| [architect](skills/architect/SKILL.md) | Technology selection, system design, deployment and test strategy; plans, delegates and reviews implementation | `architect design`, `architect impl <task>` |

Planned: `developer`, `devops`, `qa` (called by the architect).

## Flow

1. **Coordinator** researches, writes requirements (`BR-xx`, `FR-xx`, `NFR-xx`) and logs technical topics without pausing.
2. When the investigation ends, the coordinator hands a brief to the **architect**, unless the run is `-r`.
3. The **architect** designs (`docs/architecture/`), then implements tasks through the developer, devops and qa skills and opens PRs.
4. The owner is asked only for decisions that are theirs.

## Layout

```
skills/<skill-name>/SKILL.md   # plus any references/ or scripts/ the skill needs
```

## Installing

Upload a skill folder (zipped) in claude.ai under Settings → Capabilities → Skills, or copy it into `~/.claude/skills/` for Claude Code.
